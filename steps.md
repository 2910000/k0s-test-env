```
$ sudo apt update
$ sudo apt install virt-manager virtinst libvirt-daemon-system j2cli git genisoimage libnss-libvirt
$ sudo systemctl enable libvirtd
```

`/etc/nsswitch.conf`で、`hosts:`で始まる行に、"libvirt"を追加する

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
$ qemu-img create -b jammy-server-cloudimg-amd64.img -f qcow2 -F qcow2 jammy-image.img 10G
```

```
$ mkdir ~/git
$ cd ~/git
$ git clone https://github.com/2910000/k0s-test-env.git
$ mkdir ~/.ssh
$ ssh-keygen -f ~/.ssh/client_key -P ""
$ mkdir ~/out
$ export ssh_key="`cat ~/.ssh/client_key.pub`" 
$ for dir in {controller,worker1,worker2}; do \
    mkdir ~/out/$dir; \
    j2 -e ""  ~/git/k0s-test-env/user-data.j2 > ~/out/$dir/user-data; \
    name="$dir" j2 -e ""  ~/git/k0s-test-env/meta-data.j2 > ~/out/$dir/meta-data; \
    genisoimage -output ~/out/$dir/cidata.iso -V cidata -r -J ~/out/$dir/user-data ~/out/$dir/meta-data; \
    virt-install --name=$dir --ram=2048 --vcpus=1 --import --disk path=~/jammy-image-$dir.img,format=qcow2 --disk path=~/out/$dir/cidata.iso,device=cdrom --os-variant=ubuntu22.04 --network bridge=virbr0,model=virtio --noautoconsole; done
```


```
$ cat >> ~/.ssh/config << END

Host controller1 controller2 worker1 worker2
    User user
    IdentityFile ~/.ssh/client_key

END
```
