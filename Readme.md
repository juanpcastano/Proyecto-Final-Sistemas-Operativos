chmod 600 host_ssh_key

ssh -i host_ssh_key -p 2222 admin@127.0.0.1
ssh -i host_ssh_key -p 2222 tester@127.0.0.1
ssh -i host_ssh_key -p 2222 appUser@127.0.0.1

ssh-keygen -f "/home/kipi/.ssh/known_hosts" -R "[127.0.0.1]:2222"
