`bash-3.2$ # login to first host`

`bash-3.2$ ssh yasker-vm1`

`yasker@yasker-vm1:~$ # start container and create a convoy volume named vol1`

`yasker@yasker-vm1:~$ sudo docker run -it -v vol1:/vol1 --volume-driver=convoy ubuntu`

`root@fa6558ec74f2:/# # create a file in the convoy volume called /vol1/foo`

`root@fa6558ec74f2:/# touch /vol1/foo`

`root@fa6558ec74f2:/# exit`

`yasker@yasker-vm1:~$ # take a snapshot called snap1 for vol1`

`yasker@yasker-vm1:~$ sudo convoy snapshot create vol1 --name snap1`

`4311c7b5-7cc5-404c-a6d3-c11530b24099`

`yasker@yasker-vm1:~$ # backup snapshot snap1 to S3 objectstore`

`yasker@yasker-vm1:~$ sudo convoy backup create snap1 --dest s3://convoy-demo@us-west-2/`

`s3://convoy-demo@us-west-2/?backup=7a07c344-be75-4e55-87c8-81dc2a09e8c0\u0026volume=f9fae32`

`3-fbd2-4a89-b3b2-0700b5860e88`

`yasker@yasker-vm1:~$ logout`

`Connection to yasker-vm1 closed.`

`bash-3.2$ # login to another host`

`bash-3.2$ ssh yasker-vm2`

`Last login: Thu Aug 20 06:09:19 2015 from 50.255.37.17`

`[yasker@yasker-vm2 ~]$ # restore the convoy volume from backup saved in S3 objectstore`

`[yasker@yasker-vm2 ~]$ sudo convoy create vol1 --backup s3://convoy-demo@us-west-2/?backup=`

`7a07c344-be75-4e55-87c8-81dc2a09e8c0\u0026volume=f9fae323-fbd2-4a89-b3b2-0700b5860e88`

`ae1abd39-7ebf-494b-a74f-2898604baad3`

