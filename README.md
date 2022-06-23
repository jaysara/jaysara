Verify Your Environment
We've produced a verification binary that includes everything to ensure that your hosted environment will works correctly. It's built similarly to the production backend binary, using Almalinux 8.5, so it should be RHEL 8.5 compatible. The binary launches a web server and has a few HTTP endpoints to verify that your installation works as expected.
Download Integration Test Backend Binary
If Spring Labs hasn't directly provided a verification binary to you, you can download it using scp from the Sandbox Environment host
Replace USERNAME with the username provided to you.
Replace HOSTNAME with the endpoint provided to you.
scp -i ~/.ssh/spring_sandbox_ed25519 USERNAME@HOSTNAME:/srv/tokenization/poc/app /path/on/host/to/app
Host Requirements
In order to deploy the verification binary you'll need to have a host available with the following requirements:
RHEL 8.5 Compatible Linux Distribution
x64
Accessible to the network machine you're testing HTTP endpoints from
The network port is configurable in the section below
Configuration
The verification binary provided must be configured with a config.ini file accessible on the host running it.
The configuration settings will be loaded from a config.ini file in the exact file path specified with the SL_TOKENIZATION_CONFIG_PATH environment variable.
SL_TOKENIZATION_CONFIG_PATH=/path/to/config.ini
Here's an simple example of a config.ini that configures two workers and deploys a server to localhost port 8081.
[SL_TOKENIZATION_WSGI]
# Configuration variables for WSGI
# Set of available options: https://docs.gunicorn.org/en/stable/settings.html
bind = 0.0.0.0:8081
workers = 2 
​
# Debug
print_config = True
check_config = True
loglevel = Debug
Note:
You will want to configure the number of cores of your host machine to match workers 
You will also want to configure the socket/port with bind
If you want to configure specific things like host SSL termination, threads, and processses, please familiarize yourself with the documentation for Gunicorn settings. ​
Deploy Integration Test Binary to Host
To deploy the verification binary, you'll want to scp the binary that you downloaded from the sandbox environment to your host.
Replace TESTUSERNAME with the username of your given host.
Replace TESTHOSTNAME with the endpoint of your given host.
scp /path/on/host/to/app TESTUSERNAME@TESTHOST:/path/to/app
You may need to configure executable privileges for your app to whatever your environment requires.
ssh -i ~/.ssh/TESTHOSTKEY TESTUSERNAME@TESTHOST chmod 644 /path/to/app
Once you've copied your binary to your destination system, you can run your executable by connecting to your instance with ssh and running the executable at /path/to/app
Here's an example of the logs you will see while running the app on a host:
$ export SL_TOKENIZATION_CONFIG_PATH=/path/to/config.ini
$ /path/to/app
​
Running successfully
[2022-06-17 18:09:14 +0000] [7] [DEBUG] Current configuration:
[2022-06-17 18:09:14 +0000] [7] [INFO] Starting gunicorn 20.1.0
[2022-06-17 18:09:14 +0000] [7] [DEBUG] Arbiter booted
[2022-06-17 18:09:14 +0000] [7] [INFO] Listening at: http://0.0.0.0:8081 (7)
[2022-06-17 18:09:14 +0000] [7] [INFO] Using worker: sync
[2022-06-17 18:09:14 +0000] [10] [INFO] Booting worker with pid: 10
[2022-06-17 18:09:14 +0000] [11] [INFO] Booting worker with pid: 11
[2022-06-17 18:09:14 +0000] [7] [DEBUG] 2 workers
​
Running succesfully in a PyInstaller bundle
HTTP Endpoints in Integration Test 
Once your verification binary is running successfully with your configuration, you can test each HTTP endpoint on the test host from another host using curl (such as in the examples below) or whatever web browser or command line tool you may prefer like wget
Make sure to replace TESTHOST with your test hostname or IP.
TESTHOST:8000/
is the root endpoint and will log an info message
TESTHOST:8000/test_logging
will log an info message
TESTHOST:8000/test_logging_error
will log an error message
TESTHOST:8000/test
will run a full suite of tests that confirms that all of the cryptography functions needed work as expected. This endpoint will return the status of the entirety of the test suite in the response once complete. Note: Because this runs a full test suite, it may take 30+ seconds to return a response.
$ curl TESTHOST:8000/
> <p>Root endpoint successful</p>
​
$ curl TESTHOST:8000/test_logging
> <p>This should log an info message</p>
​
$ curl TESTHOST:8000/test_logging_error
> <p>This should log an error message</p>
​
$ curl TESTHOST:8000/test
> <p>All tests were collected and passed successfully</p>
On the running host, you should see something similar to these logs below.
Root endpoint successful
​
[2022-06-17 18:10:24 +0000] [11] [DEBUG] GET /test_logging
This should log an info message
​
[2022-06-17 18:10:34 +0000] [11] [DEBUG] GET /test_logging_error
This should log an error message
​
[2022-06-17 18:13:28 +0000] [10] [DEBUG] GET /test
​
...
============================= test session starts ==============================
...
================= 231 passed, 65 skipped, 42 warnings in 5.22s =================
Test endpoint successful
All tests were collected and passed successfully
​
Verification
If everything worked successfully and there were no unexpected response codes or errors in the logs on the host, congratulations, you have everything you need to work with the backend on-prem deployment.
If there was an error with your deployment of the test binary, please share complete logs as well as any relevant system information with the Spring Labs team via email.
Here's a shell script you can run on your host to share system information.
#!/bin/bash
echo -e "-------------------------------System Information----------------------------"
echo -e "Hostname:\t\t"`hostname`
echo -e "uptime:\t\t\t"`uptime | awk '{print $3,$4}' | sed 's/,//'`
echo -e "Manufacturer:\t\t"`cat /sys/class/dmi/id/chassis_vendor`
echo -e "Product Name:\t\t"`cat /sys/class/dmi/id/product_name`
echo -e "Version:\t\t"`cat /sys/class/dmi/id/product_version`
echo -e "Machine Type:\t\t"`vserver=$(lscpu | grep Hypervisor | wc -l); if [ $vserver -gt 0 ]; then echo "VM"; else echo "Physical"; fi`
echo -e "Operating System:\t"`hostnamectl | grep "Operating System" | cut -d ' ' -f5-`
echo -e "Kernel:\t\t\t"`uname -r`
echo -e "Architecture:\t\t"`arch`
echo -e "Processor Name:\t\t"`awk -F':' '/^model name/ {print $2}' /proc/cpuinfo | uniq | sed -e 's/^[ \t]*//'`
echo -e "Active User:\t\t"`w | cut -d ' ' -f1 | grep -v USER | xargs -n1`
echo -e "System Main IP:\t\t"`hostname -I`
echo ""
echo -e "-------------------------------CPU/Memory Usage------------------------------"
echo -e "Memory Usage:\t"`free | awk '/Mem/{printf("%.2f%"), $3/$2*100}'`
echo -e "CPU Usage:\t"`cat /proc/stat | awk '/cpu/{printf("%.2f%\n"), ($2+$4)*100/($2+$4+$5)}' |  awk '{print $0}' | head -1`
echo ""
When providing system information using the above script, it should look like this:
-------------------------------System Information----------------------------
Hostname:		ip-x-x-x-x.us-west-2.compute.internal
uptime:			13 days
Manufacturer:		Amazon EC2
Product Name:		m5.large
Version:
Machine Type:		VM
Operating System:	AlmaLinux 8.6 (Sky Tiger)
Kernel:			4.18.0-372.9.1.el8.x86_64
Architecture:		x86_64
Processor Name:		Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz
Active User:		example
System Main IP:		10.1.1.101
​
-------------------------------CPU/Memory Usage------------------------------
Memory Usage:	6.19%
CPU Usage:	0.58%
​
