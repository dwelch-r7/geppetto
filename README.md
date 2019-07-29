# Geppetto

## About
Geppetto is an infrastructure automation tools used to orchestrate the state of virtual machines for testing deployed binary files in a repeatable way. While the original intended use case is specific to the [Metasploit Framework Project](https://metasploit.com) for use in testing deliverable payloads against virtual machines and physical hardware, the resulting tool has been able to play puppeteer for multiple scenarios.

## Purpose
The idea behind this project is to create an automated testing framework that can test metasploit payloads and exploits against actual targets.  It takes in json config files containing target data, exploit data, and payload data, then creates bash and/or python configuration scripts for the hists and rc scripts for msfconsole sessions that it uploads and runs on host machines as-needed.  This project was created originally with the expectation it would use a sister project, [vm-automation](https://github.com/rapid7/vm-automation), but with increasing functionality, the requirement that targets be virtualized is gone, and hopefully future improvements will allow the msfconsole sessions to run on physical hardware as well.

### How does it work?
Originally, this script was targeted to virtual machines and exclusively for reverse `/exploit/multi/handler` "exploits" so that's a good place to start the explination:
To test a payload with an `exploit/multi/handler` listener, we need to create the payload on a framework VM (__MSF_HOST__), start a listener on the __MSF_HOSTS__, upload the payload to the VM we're testing (__TARGET__) and start the payload.  Once we get the callback and establish the session, we nee to execute commands (__COMMAND_LIST__) and then somehow figure out if it worked.
That got a bit complicated when we added support for bind payloads, as it meant that slfconsole needed to get launched on after the payloads was uploaded and started on the __TARGET__.  Supporting exploits was comparatively easy, but to make some sense of how it all works, I've kind of split things into three phases:
* Stage One: Configure __MSF_HOST__:
  * Create the binary payload if the exploit is `exploit/multi/handler`
  * Create an rc script to set up and start the msfconsole session
  * If the payload is a _reverse_ payload or an rce exploit, start msfconsole with the rc script
* Stage Two: Configure __TARGETS__:
  * Create python or bash scripts to download `/exploit/multi/handler` payloads and launch them
* Stage Three: Start Bind sessions on __MSF_HOSTS__
  * If the payload is a _bind_ payload, start msfconsole with the rc script
  
  It is not hard to see that the only time all three stages are required are with a bind payload and a `exploit/multi/handler` "exploit."  Everything else only needs one or two of the stages, but it makes life easier for me to separate the logic into these three stages.
  
### What can it do now?
Currently, the test framework _should_ support all IPv4 payloads and exploits.  IPv6 is probably achieveable with moderate difficulty, but I desperately want to avoid using IPv6 until I get a colon key on my numpad.  

TBH, I've also not tested with a lot of the more complex payloads, as I've no idea what some of them do.  I have tested with the following payloads (There is some trivial cleanup required before mettle payloads are supported with `/exploit/multi/handler`):
* `windows/meterpreter/bind_tcp`
* `windows/meterpreter/bind_tcp_rc4`
* `windows/meterpreter/bind_tcp_uuid`
* `windows/meterpreter/reverse_http`
* `windows/meterpreter/reverse_https`
* `windows/meterpreter/reverse_tcp`
* `windows/meterpreter/reverse_tcp_dns`
* `windows/meterpreter/reverse_tcp_rc4`
* `windows/meterpreter/reverse_tcp_rc4_dns`
* `windows/meterpreter/reverse_tcp_uuid`
* `windows/meterpreter/reverse_winhttp`
* `windows/meterpreter/reverse_winhttps`
* `windows/meterpreter_bind_tcp`
* `windows/meterpreter_reverse_http`
* `windows/meterpreter_reverse_https`
* `windows/meterpreter_reverse_tcp`
* `windows/x64/meterpreter/bind_tcp`
* `windows/x64/meterpreter/bind_tcp_uuid`
* `windows/x64/meterpreter/reverse_http`
* `windows/x64/meterpreter/reverse_https`
* `windows/x64/meterpreter/reverse_tcp`
* `windows/x64/meterpreter/reverse_tcp_uuid`
* `windows/x64/meterpreter/reverse_winhttp`
* `windows/x64/meterpreter/reverse_winhttps`
* `windows/x64/meterpreter_bind_tcp`
* `windows/x64/meterpreter_reverse_http`
* `windows/x64/meterpreter_reverse_https`
* `windows/x64/meterpreter_reverse_tcp`
* `linux/mipsle/meterpreter/reverse_tcp`
* `linux/x64/meterpreter/bind_tcp`
* `linux/x64/meterpreter/reverse_tcp`
* `linux/x86/meterpreter/bind_tcp`
* `linux/x86/meterpreter/bind_tcp_uuid`
* `linux/x86/meterpreter/reverse_tcp`
* `linux/x86/meterpreter/reverse_tcp_uuid`
* `java/meterpreter/bind_tcp`
* `java/meterpreter/reverse_http`
* `java/meterpreter/reverse_https`
* `java/meterpreter/reverse_tcp`
* `python/meterpreter/bind_tcp`
* `python/meterpreter/bind_tcp_uuid`
* `python/meterpreter/reverse_http`
* `python/meterpreter/reverse_https`
* `python/meterpreter/reverse_tcp`
* `python/meterpreter/reverse_tcp_ssl`
* `python/meterpreter/reverse_tcp_uuid`
* `python/meterpreter_bind_tcp`
* `python/meterpreter_reverse_http`
* `python/meterpreter_reverse_https`
* `python/meterpreter_reverse_tcp`


  
### json config files
Here are the values found in the json config files:
* __TEST_NAME__: This is just the name for the test.  It can be whatever you want it to be, and only appears inside the reports generated by the scripts.
* __REPORT_PREFIX__: This value is paired with a timestamp to prefix report filenames and directories generated during the test.
* __GIT_BRANCH__: The upstream Framework branch you wish to use when testing.  It may be worthwhile to allow adding other repos later, but rught now, this will only accept branches located in upstream Framework.
* __HTTP_PORT__: in the case of manually-crafted payloads (if you use `exploit/multi/handler`), this is the port that the http server will use to serve out the payload files that are created by msfvenom.
* __STARTING_LISTENER__: The script requires a lot of unique, ephemeral ports to use in payload creation and exploit callbacks.  That is accomplished by incremeneting this value each time a unique port is required.  This will be the lowest port value used.
* __MSF_HOSTS__: A list of internet-connected linux VMs with Framework installed to the path `/home/<user>/rapid7/metasploit-framework/`
  * __TYPE__: [CURRENTLY, THE ONLY SUPPORTED TYPE FOR MSF_HOSTS IS `VIRTUAL`] The type of target, your choices are `VIRTUAL` if you would like to leverage the vm-automation library functions or `PHYSICAL` if you are just going to assume that the VM is up, running, and in the right state when the test executes.
  * __METHOD__: [CURRENTLY, THE ONLY SUPPORTED METHOD FOR MSF_HOSTS IS `VM_TOOLS_UPLOAD`] The way you expect the session to start.  Right now, the two values available are `VM_TOOLS_UPLOAD` if you want to use an exploit like `/exploit/multi/handler` and have the script upload teh payload using vmware tools and `EXPLOIT` if you are going to use a good old RCE exploit.
  * __HYPERVISOR_CONFIG__: Another json file with the information on the hypervisor used.  That's defined in the vm-automation documentation and required if you are using vm-automation.
  * __NAME__: The VM name as it appears in the hypervisor if you are leveraging vm-automation or the name as you want it to appear in reports if it is not using vm-automation.
  * __USERNAME__: The username for the VM (Required if you are using `VM_TOOLS_UPLOAD`)
  * __PASSWORD__: The password for the VM (Required if you are using `VM_TOOLS_UPLOAD`)
  * __PAYLOAD_DIRECTORY__: The directory you want to store the payloads (Required if you are using `/exploit/multi/handler`)
  * __PYTHON_PATH__: Required if you are using python payloads.  It is the full path to the python binary you want to use for testing python payloads.  If python is stored in the system's path variable, this can be simply `python`.  Using this variable allows you to install several version of python on the same VM, and target different versions as needed.
  * __JAVA_PATH__: Required if you are using java payloads.  It is the full path to the java binary you want to use for testing java payloads.  If java is stored in the system's path variable, this can be simply `java`.  Using this variable allows you to install several version of java on the same VM, and target different versions as needed.
  * __TESTING_SNAPSHOT__: This is he snapshot to use for testing.  You can  specfy different snapshots for different tests.  This is also the snapshot that the script will revert the VM back to after testing is complete.
  * __PAYLOADS__: This allows you to specify payloads for _this particular target_.  There is a top-level payload value that allows you to specify payloads to use on all targets.  It has two values, __NAME__ and __SETTINGS__.  __NAME__ is the exploit name as it appears in Framework, __SETTINGS__ is a list of settings in msfvenom-style format, i.e. `RC4PASSWORD=secret` or `SMBUser=Administrator`.  To make life easier, you can specify that you would like a unique port by using the keyword `UNIQUE_PORT`: For example, if you put `LPORT=UNIQUE_PORT`, the script would replace the string `UNIQUE PORT` with a non-repeating port based off of the `STARTING_LISTENER` value specified above.
  * __EXPLOITS__: This allows you to specify exploits for _this particular target_.  There is a top-level expoloit value that allows you to specify payloads to use on all targets.  Just like __PAYLOAD__ above, it has two subvalues: `NAME` and `SETTINGS`.  Name is the name of the payload as used by Framework, and settings are msfvenom-style strings that specify options for the exploit.  The `UNIQUE_PORT` keyword is supported in __EXPLOIT__ settings just as it is in PAYLOAD settings.
  
* __TARGETS__- A list of targets that can be virtual or physical, with multiple options (coming) for interaction.
  * __TYPE__: Same as __MSF_HOSTS__, but __TARGETS__ will support either `VIRTUAL` or `PHYSICAL`
  * __METHOD__: Same as MSF_HOSTS, but __TARGETS__ will support either `VM_TOOLS_UPLOAD` or `EXPLOIT`
  * __HYPERVISOR_CONFIG__: Same as __MSF_HOSTS__
  * __NAME__: Same as __MSF_HOSTS__
  * __USERNAME__: Same as __MSF_HOSTS__
  * __PASSWORD__: Same as __MSF_HOSTS__
  * __PAYLOAD_DIRECTORY__: The directory you want to store the payloads (Required if you are using `/exploit/multi/handler`)
  * __PYTHON_PATH__: Required if you are using python payloads.  It is the full path to the python binary you want to use for testing python payloads.  If python is stored in the system's path variable, this can be simply `python`.  Using this variable allows you to install several version of python on the same VM, and target different versions as needed.
  * __JAVA_PATH__: Required if you are using java payloads.  It is the full path to the java binary you want to use for testing java payloads.  If java is stored in the system's path variable, this can be simply `java`.  Using this variable allows you to install several version of java on the same VM, and target different versions as needed.
  * __TESTING_SNAPSHOT__: This is he snapshot to use for testing.  You can  specfy different snapshots for different tests.  This is also the snapshot that the script will revert the VM back to after testing is complete.
  * __PAYLOADS__: This allows you to specify payloads for _this particular target_.  There is a top-level payload value that allows you to specify payloads to use on all targets.  It has two values, __NAME__ and __SETTINGS__.  NAME is the exploit name as it appears in Framework, __SETTINGS__ is a list of settings in msfvenom-style format, i.e. `RC4PASSWORD=secret` or `SMBUser=Administrator`.  To make life easier, you can specify that you would like a unique port by using the keyword `UNIQUE_PORT`: For example, if you put `LPORT=UNIQUE_PORT`, the script would replace the string `UNIQUE PORT` with a non-repeating port based off of the `STARTING_LISTENER` value specified above.
  * __EXPLOITS__: This allows you to specify exploits for _this particular target_.  There is a top-level expoloit value that allows you to specify payloads to use on all targets.  Just like __PAYLOAD__ above, it has two subvalues: `NAME` and `SETTINGS`.  Name is the name of the payload as used by Framework, and settings are msfvenom-style strings that specify options for the exploit.  The `UNIQUE_PORT` keyword is supported in __EXPLOIT__ settings just as it is in PAYLOAD settings.
* __PAYLOADS__ (OPTIONAL-ISH)- This is a global way to specify payloads.  You can specify the payload inside the TARGETS value, and it will be applied only to that specific target, but any payloads listed in teh top-level PAYLOADS value will be applied to all targets, with some very basic logic to try and prevent unsupported payloads from getting sent to targets.  The logic relies on how you name the VMs, so please follow the naming conventions. This is OPTIONAL-ISH as it makes sense to have at least one payload specified for each target or else, it will not be tested, but the script does not really care; if no payload is specified for a target, then nothing will happen.
* __EXPLOITS__ (OPTIONAL-ISH)- Just like the global PAYLOADS value, this is a global EXPLOITS value.  Exploits listed here will be applied to all targets, without _any_ attempt to prevent exokiuts from being used against unsupported targets.
* __COMMAND_LIST__- A list of commands to execute after establishing a session
* __SUCCESS_LIST__- A list of strings that the session must contain for the session to be considered a success

## Naming your targets
Yeah, not thrilled about this solution in a lot of ways, but I think it is acceptable, as it helps some, and we should use comprehensible names for the VMs we use, anyway, and these three rules will help out a lot so we can use global payloads.  
Rather than post my guidelines for VM naming, I'm just going to explain the basic logic present that tries to prevent usupported payloads (All string matching is case insensitive):
* A payload continaing _win_ will not be used unless _win_ is present in the VM name.
* A payload containing _mettle_ will not be used if _win_ is in the VM name.
* A payload containing _x64_ will not be used unless _x64_ is present in the VM name.
So to be clear, if it is a Windows VM, make sure 'win' is in the name and if it is a 64-bit OS, make sure 'x64' is present in the name.  That's it; and for that requirement, global payloads will work much better.

## Artifacts created by the tests
The test creates a new directory in a directory called test_data.  The directory it creates will be `<test_name>_<timestamp>` so that you'll have some high-level ability to determine what's in the directory, but the directory will not get overwritten by subsequent runs.
Within that directory, there will be three directories:
* `reports` will contain teh reports generated during the testing, including:
  * `commit_<ip>.txt` will be the framework commit versions on each MSF_HOST
  * `test.json` will contain the output json with global payloads and values written
  * `testlog.log` contains the log data from teh test itself for debugging purposes (currently this is also sent to stdout)
  * <reportname>.html will be a really simple html document that lists TARGET, MSF_HOST, PAYLOAD, EXPLOIT, and success, with a link to the text of the session that was (hopefully) generated.
* `scripts` contains the scripts generated to support the tests and kept to aid debugging.
* `sessions` contains the rc scripts sent to Framework and the text of the session (hopefully) generated.  The sessions are linked from the report.html file.

## Executing tests with Docker
```
cd geppetto/docker
docker build -t rapid7/build:geppetto .
docker run --rm=true --tty -u jenkins \
    --volume=${PATH_TO_WORKING_DIR}:/r7-source \
    --workdir=/r7-source/geppetto rapid7/build:geppetto \
    bash -l -c "python autoPayloadTest.py ${DOCKER_RELATIVE_PATH_TO_TEST_JSON}"
```
