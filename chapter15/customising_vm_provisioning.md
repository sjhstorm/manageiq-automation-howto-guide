## Example - Customising VM Provisioning

### Scenario

We are using a RHEV Provider with our CloudForms installation, and we can successfully provision VMs using **Native Clone** provision type from fully configured RHEV Templates. The Templates all have a single 30GB thin-provisioned hard drive.

### Task

We would like all VMs provisioned from these Templates to have a second 30GB hard drive added automatically during provisioning. The second drive should be created in the same RHEV Storage Domain as the first drive, (i.e. not hard-coded to a Storage Domain).

### Methodology

Edit the `VMProvision_VM` State Machine to add two new States to perform the task. We'll add the second disk using the RHEV RESTful API, using credentials stored for the Provider.

#### Step 1

The first thing we weed to do is clone the `ManageIQ/Infrastructure/VM/Provisioning/StateMachines/VMProvision_VM/Provision VM from Template (template)` State Machine Instance into our own `ACME` Domain so that we can edit the schema:
<br> <br>

![screenshot](images/screenshot12.png)

<br>
Now we edit the schema of the copied Class:
<br> <br>

![screenshot](images/screenshot13.png)

<br>
...and add two more steps, **AddDisk** and **StartVM**
<br> <br>

![screenshot](images/screenshot14.png)

Adjust the Class Schema Sequence so that the steps come after **PostProvision**:
<br> <br>

![screenshot](images/screenshot15.png)

#### Step 2

We're going to override the default behaviour of the VM Provisioning workflow which is to auto-start a VM after provisioning. We do this because we want to add our new disk with the VM powered off, and then power on the VM ourselves afterwards.

We clone the `ManageIQ/Infrastructure/VM/Provisioning/StateMachines/Methods/redhat_CustomizeRequest` Method into our Domain:
<br> <br>

![screenshot](images/screenshot16.png?)

<br>
We edit `redhat_CustomizeRequest` to set the Options Hash key `:vm_auto_start` to be ```false```:
<br> <br>

```ruby
#
# Description: This method is used to Customize the RHEV, RHEV PXE, 
# and RHEV ISO Provisioning Request
#

# Get provisioning object
prov = $evm.root["miq_provision"]

# Set the autostart parameter to false so that RHEV won't start the VM directly
$evm.log(:info, "Setting vm_auto_start to false")
prov.set_option(:vm_auto_start, [false, 0])

$evm.log("info", "Provisioning ID:<#{prov.id}> \
Provision Request ID:<#{prov.miq_provision_request.id}> \
Provision Type: <#{prov.provision_type}>")


```
#### Step 3
We need to add two new Instances `AddDisk` and `StartVM`, and two new Methods `add_disk` and `start_vm`. Add the corresponding Method names to the **execute** schema field of each Instance:
<br> <br>

![screenshot](images/screenshot17.png)

<br>
The code for **add_disk** is as follows...
<br> <br>


```ruby
#------------------------------------------------------------------------------
#
# CFME Automate Method: add_disk
#
# Authors: Kevin Morey, Peter McGowan (Red Hat)
#
# Notes: This method adds a disk to a RHEV VM
#
#------------------------------------------------------------------------------

require 'rest_client'
require 'nokogiri'

NEW_DISK_SIZE = 30
@debug = false

begin
  
  #------------------------------------------------------------------------------
  def call_rhev(servername, username, password, action, ref=nil, body_type=:xml, body=nil)
    #
    # If ref is a url then use that one instead
    #
    unless ref.nil?
      url = ref if ref.include?('http')
    end
    url ||= "https://#{servername}#{ref}"
    
    params = {
      :method => action,
      :url => url,
      :user => username,
      :password => password,
      :headers => { :content_type=>body_type, :accept=>:xml },
      :verify_ssl => false
    }
    params[:payload] = body if body
    if @debug
      $evm.log(:info, "Calling RHEVM at: #{url}")
      $evm.log(:info, "Action: #{action}")
      $evm.log(:info, "Payload: #{params[:payload]}")
    end
    rest_response = RestClient::Request.new(params).execute
    #
    # RestClient raises an exception for us on any non-200 error
    #
    return rest_response
  end
  #------------------------------------------------------------------------------

  #------------------------------------------------------------------------------
  # Start of main code
  #
  case $evm.root['vmdb_object_type']
  when 'miq_provision'                  # called from a VM provision workflow
    vm = $evm.root['miq_provision'].destination
    disk_size_bytes = NEW_DISK_SIZE * 1024**3
  when 'vm'
    vm = $evm.root['vm']                # called from a button
    disk_size_bytes = $evm.root['dialog_disk_size_gb'].to_i * 1024**3
  end
  
  storage_id = vm.storage_id rescue nil
  $evm.log(:info, "VM Storage ID: #{storage_id}") if @debug
  #
  # Extract the RHEV-specific Storage Domain ID
  #
  unless storage_id.nil? || storage_id.blank?
    storage = $evm.vmdb('storage').find_by_id(storage_id)
    storage_domain_id = storage.ems_ref.match(/.*\/(\w.*)$/)[1]
    if @debug
      $evm.log(:info, "Found Storage: #{storage.name}")
      $evm.log(:info, "ID: #{storage.id}")
      $evm.log(:info, "ems_ref: #{storage.ems_ref}") 
      $evm.log(:info, "storage_domain_id: #{storage_domain_id}")
    end
  end

  unless storage_domain_id.nil?
    #
    # Extract the IP address and credentials for the RHEV Provider
    #
    servername = vm.ext_management_system.ipaddress || vm.ext_management_system.hostname
    username = vm.ext_management_system.authentication_userid
    password = vm.ext_management_system.authentication_password

    builder = Nokogiri::XML::Builder.new do |xml|
      xml.disk {
        xml.storage_domains {
          xml.storage_domain :id => storage_domain_id
        }
        xml.size disk_size_bytes
        xml.type 'system'
        xml.interface 'virtio'
        xml.format 'cow'
        xml.bootable 'false'
      }
    end

    body = builder.to_xml
    $evm.log(:info, "Adding #{disk_size_bytes / 1024**3} GByte disk to VM: #{vm.name}")
    response = call_rhev(servername, username, password, :post, "#{vm.ems_ref}/disks", :xml, body)
    #
    # Parse the response body XML
    #
    doc = Nokogiri::XML.parse(response.body)
    #
    # Pull out some re-usable href's from the initial response
    #
    disk_href = doc.at_xpath("/disk")['href']
    creation_status_href = doc.at_xpath("/disk/link[@rel='creation_status']")['href']
    activate_href = doc.at_xpath("/disk/actions/link[@rel='activate']")['href']
    if @debug
      $evm.log(:info, "disk_href: #{disk_href}")
      $evm.log(:info, "creation_status_href: #{creation_status_href}")
      $evm.log(:info, "activate_href: #{activate_href}")
    end
    #
    # Validate the creation_status (wait for up to a minute)
    #
    creation_status = doc.at_xpath("/disk/creation_status/state").text
    counter = 13
    $evm.log(:info, "Creation Status: #{creation_status}")
    while creation_status != "complete"
      counter -= 1
      if counter == 0
        raise "Timeout waiting for new disk creation_status to reach \"complete\": \
               Creation Status = #{creation_status}"
      else
        sleep 5
        response = call_rhev(servername, username, password, :get, creation_status_href, :xml, nil)
        doc = Nokogiri::XML.parse(response.body)
        creation_status = doc.at_xpath("/creation/status/state").text
        $evm.log(:info, "Creation Status: #{creation_status}")
      end
    end
    #
    # Disk has been created successfully,
    # now check its activation status and if necessary activate it
    #
    response = call_rhev(servername, username, password, :get, disk_href, :xml, nil)
    doc = Nokogiri::XML.parse(response.body)
    if doc.at_xpath("/disk/active").text != "true"
      $evm.log(:info, "Activating disk")
      body = "<action/>"
      response = call_rhev(servername, username, password, :post, activate_href, :xml, body)
    else
      $evm.log(:info, "New disk already active")
    end
  end
  #
  # Exit method
  #
  $evm.root['ae_result'] = 'ok'
  exit MIQ_OK
  #
  # Set Ruby rescue behavior
  #
rescue RestClient::Exception => err
  $evm.log(:error, "The REST request failed with code: #{err.response.code}") unless err.response.nil?
  $evm.log(:error, "The response body was:\n#{err.response.body.inspect}") unless err.response.nil?
  $evm.root['ae_reason'] = "The REST request failed with code: #{err.response.code}" unless err.response.nil?
  $evm.root['ae_result'] = 'error'
  exit MIQ_STOP
rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  $evm.root['ae_reason'] = "Unspecified error, see automation.log for backtrace"
  $evm.root['ae_result'] = 'error'
  exit MIQ_STOP
end
```
<br>
The code for **start_vm** is as follows:
<br> <br>

```ruby
#----------------------------------------------------------------
#
# CFME Automate Method: start_vm
#
# Author: Peter McGowan (Red Hat)
#
#----------------------------------------------------------------
begin
  vm = $evm.root['miq_provision'].destination
  $evm.log(:info, "Current VM power state = #{vm.power_state}")
  unless vm.power_state == 'on'
    vm.start
    vm.refresh
    $evm.root['ae_result'] = 'retry'
    $evm.root['ae_retry_interval'] = '30.seconds'
  else
    $evm.root['ae_result'] = 'ok'
  end

rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  $evm.root['ae_result'] = 'error'
end
```

The scripts are also available [here](https://github.com/pemcg/cloudforms-automation-howto-guide/tree/master/chapter15/scripts)

#### Step 4

Now we edit our copied `Provision VM from Template` State Machine Instance to add the `AddDisk` and `StartVM` Instance URIs to the appropriate steps:
<br> <br>

![screenshot](images/screenshot18.png?)

#### Step 5

Provision a VM. We should see the the VM is not immediately started after provisioning, and
suitable messages in automation.log showing that our additonal Methods are working:
<br> <br>

```
...<AEMethod add_disk> Adding 30GB disk to VM: rhel7srv006
...<AEMethod add_disk> Creation Status: pending
...<AEMethod add_disk> Creation Status: complete
...<AEMethod add_disk> New disk already active
...
...<AEMethod start_vm> Current VM power state = off
...<AEMethod start_vm> Current VM power state = unknown
...<AEMethod start_vm> Current VM power state = on
```
<br>
If we look at the number of disks in the VM Details page in CloudForms, we see:
<br> <br>

![screenshot](images/screenshot19.png)
<br>

Success!
