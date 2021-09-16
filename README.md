# Create Calico Cloud-compatible cluster in Rancher
Create a EKS cluster through Rancher - which is compatible for Calico Cloud

## Create a compatible EC2 instance:
Make sure the SSH key was given correct permissions
```
sshkeys % chmod 400 nigel-rancher-key.pem
```
SSH into the newly-created EC2 instance
```
ssh -i "nigel-rancher-key.pem" ec2-user@192.168.185.196
```

## Install Rancher
Check Docker Status
```
sudo systemctl status docker
```

Enable Docker runtime
```
sudo systemctl enable docker
```

Start the Docker Runtime
```
sudo systemctl start docker
```

```
sudo /usr/sbin/usermod -aG docker $USER
```

```
sudo chown $USER:docker /var/run/docker.sock
```

This ```DID NOT WORK``` after original test:
```
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

This ```DID WORK``` after adding the privleged flag:
```
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

Find the container ID - ace724ca4640
```
docker ps
```

Find the bootstrap password required for login:
```
docker logs  ace724ca4640  2>&1 | grep "Bootstrap Password:"
```

<img width="779" alt="Screenshot 2021-09-16 at 16 15 45" src="https://user-images.githubusercontent.com/82048393/133640590-58bf23e6-b459-4bbd-93de-b79cb2ff3333.png">
