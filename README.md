# Ceph Installation With Cephadm (Pacific Version )
<b>1-Config Ssh For Connection Between Servers (Do it on all your servers )</b> <br>
vi /etc/ssh/sshd_config <br>
Permitrootlogin --> yes <br>
Paaswordauthentication --> yes <br>
restart sshd service <br>
<b>2-Install Cephadm  (Do it on all your servers )</b><br>
curl --silent --remote-name --location curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm <br>
chmod +x cephadm <br>
cephadm --version (needs Docker Installation) <br>
After Install Docker we can add below registery For resolving Image restriction <br>
vi /etc/docker/deamon.json <br>
```yaml
{ 
  "registery-mirrors":[ 
    "https://repo.ficld.ir" 
  ], <br>
  "insecure-registries":[], 
  "debug":false,
  "log-deriver":"local", 
  "log-opts":{ 
     "max-size":"500M", 
     "max-file":"3", 
  } 
} 
```
systemctl restart docker <br>
cephadm --version <br>
cephadm add -repo --release pacific <br>
ceph install <br>
ceph bootstrap --mon-ip SERVERIP (Create Config File in /etc/ceph) <br>
<b>3-Shell Access </b><br> 
cephadm shell <br>
ceph -s (Cluster status)<br>
<b>4-Authorized servers </b><br>
copy public key from /etc/ceph/ceph.pub and paste it into /root/.ssh/authorized_key <br>
all hostnames should be in /etc/hosts of All servers <br>
