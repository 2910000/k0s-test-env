
```
$ sudo apt update
$ sudo apt install virt-manager virtinst libvirt-daemon-system j2cli git genisoimage libnss-libvirt ansible ansible-core apt-transport-https ca-certificates curl
$ sudo systemctl enable libvirtd
```

[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

```
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
$ sudo apt update
$ sudo apt install kubectl
```

`/etc/nsswitch.conf`で、`hosts:`で始まる行に、"libvirt_guest"を追加する

```
# cat >> /etc/libvirt/libvirtd.conf << END
auth_unix_ro = "none"
auth_unix_rw = "none"
unix_sock_group = "sudo"
unix_sock_ro_perms = "0777"
unix_sock_rw_perms = "0770"
END
# cat >> /etc/libvirt/qemu.conf << END
user = "user"
group = "libvirt"
END
$ sudo systemctl restart libvirtd
```

https://sumit-ghosh.com/posts/create-vm-using-libvirt-cloud-images-cloud-init/ から

```
$ wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

```
$ mkdir ~/git
$ cd ~/git
$ git clone https://github.com/2910000/k0s-test-env.git
$ mkdir ~/.ssh
$ ssh-keygen -f ~/.ssh/client_key -P ""
$ mkdir ~/out
$ export MY_SSH_KEY="`cat ~/.ssh/client_key.pub`" 
$ for dir in {controller1,controller2,worker1,worker2}; do \
    qemu-img create -b ~/jammy-server-cloudimg-amd64.img -f qcow2 -F qcow2 ~/jammy-image-$dir.img 10G; \
    mkdir ~/out/$dir; \
    j2 -e ""  ~/git/k0s-test-env/user-data.j2 > ~/out/$dir/user-data; \
    name="$dir" j2 -e ""  ~/git/k0s-test-env/meta-data.j2 > ~/out/$dir/meta-data; \
    genisoimage -output ~/out/$dir/cidata.iso -V cidata -r -J ~/out/$dir/user-data ~/out/$dir/meta-data; \
    virt-install --name=$dir --ram=4096 --vcpus=1 --import --disk path=~/jammy-image-$dir.img,format=qcow2 --disk path=~/out/$dir/cidata.iso,device=cdrom --os-variant=ubuntu22.04 --network bridge=virbr0,model=virtio --noautoconsole; done
```


```
$ cat >> ~/.ssh/config << END

Host controller1 controller2 worker1 worker2
    User user
    IdentityFile ~/.ssh/client_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

END
```

https://docs.k0sproject.io/stable/examples/ansible-playbook/

```
git clone https://github.com/movd/k0s-ansible.git
cd k0s-ansible
```

`inventory/inventory.yml`ファイルを作成:

```yaml
---
all:
  children:
    initial_controller:
      hosts:
        controller1:
    controller:
      hosts:
        controller2:
    worker:
      hosts:
        worker1:
        worker2:
  vars:
    ansible_user: user
```

Test inventory:

```
$ ansible -m ping all
```

Install k0s:

```
$ ansible-playbook site.yml
```

kubectl config:

```
$ readlink -f inventory/artifacts/k0s-kubeconfig.yml
$ export KUBECONFIG=<上記のリターン値>
```

この"export "で始まる行を`~/.bashrc`に追加すれば、セッションに入れば既に紐付けされる

上記の`k0s-kubeconfig.yml`ファイルを編集し、`server:`で始まる行を、`controller1`からcontroller1のIPアドレスに変更する

確認：

```
$ kubectl get nodes
```


```
$ for dir in {controller1,controller2,worker1,worker2}; do virsh destroy $dir && virsh undefine $dir; done
```

