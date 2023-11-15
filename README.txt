Migrate Amazon EC2 Instances to OCI Compute Instance

1. On AWS Instance, create a sample file:
echo 'Created on AWS by nishrao' >> test.txt

2. Enable Serial Console Access (https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/enablingserialconsoleaccess.htm)
	i. For Amazon Linux:
		- Configure Boot Loader & OS:
			sudo vim /etc/default/grub (**only changes listed)
				GRUB_TERMINAL="serial console"
				GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200"
				GRUB_CMDLINE_LINUX="console=tty1 console=ttyS0,115200"
			sudo grub2-mkconfig -o /boot/grub2/grub.cfg
			sudo cat /etc/securetty | grep ttyS0
			ps aux | grep ttyS0
		- Configure Networking:
			sudo yum install -y libvirt qemu-kvm virt-install virt-viewer
			sudo dracut -v -f --add-drivers "virtio virtio_pci virtio_scsi virtio_ring" /boot/initramfs-$(uname -r).img $(uname -r)
			sudo lsinitrd /boot/initramfs-$(uname -r).img | grep -i virtio
			sudo mkdir /etc/netplan
			sudo vim /etc/netplan/config.yaml
				network:
				  version: 2
				  ethernets:
					ens3:
					  dhcp4: true
			sudo vim /etc/cloud/cloud.cfg.d/01_network.cfg
				network:
				version: 2
				ethernets:
				ens*:
				dhcp4: true
	ii. For Ubuntu:
		- Configure Boot Loader & OS:
			sudo su -
			sudo grub-install --version (**execute following steps only if version is 2.xx, else follow OCI documentation, link above)
			sudo vim /etc/default/grub (**only changes listed)
				GRUB_CMDLINE_LINUX="console=tty1 console=ttyS0,115200"
				GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200"
				GRUB_TERMINAL="serial console"
			update-grub
			sudo grub-mkconfig -o /boot/grub/grub.cfg
			sudo vi /etc/securetty (**add ttyS0 at the end if not present)
				ttyS0
			ps aux | grep ttyS0

3. Shut down the instance. Note the instance-id of the instance.

4. Create an S3 bucket with ACL enabled.
	In bucket properties, edit the ACL and add an entry to "Access for other AWS account". Enable all options.
	Take canonical ID from this document: https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport.html

4. Open AWS Cloud Shell and execute the following command:
	- Create an Instance Export Task:
		aws ec2 create-instance-export-task --instance-id i-0198e311ba748ee85 --description "Export-to-OCI" --target-environment vmware --export-to-s3-task DiskImageFormat=vmdk,ContainerFormat=ova,S3Bucket=nishraoexportbucket1
	- List the Export Task Details:
		aws ec2 describe-export-tasks --export-task-ids export-i-03e802cef55cb8ec8
	- Wait for the state of the task to be "completed"
	
5. Prepare the credentials on both AWS & OCI
	- Get AWS Access & Secret Key
	- Get OCI Customer Access & Secret Key
	- Build the S3 endpoint URL for OCI bucket:
		https://<namespace>.compat.objectstorage.<region_id>.oraclecloud.com

6. Rclone the file from OCI to AWS:
	- Install rclone:
		sudo yum install -y rclone
	- Setup the config file:
		vim ~/.rclone.conf
		
		[aws]
		type=s3
		region=us-east-1
		access_key_id=
		secret_access_key=
		
		[oci]
		type=s3
		region=ap-hyderabad-1
		access_key_id=
		secret_access_key=
		endpoint=https://id03wiznwkrx.compat.objectstorage.ap-hyderabad-1.oraclecloud.com
	- Execute the rclone command (will do multipart upload):
		rclone copy aws:nishraobucket/export-i-0ada9ed19e8b773de.ova oci:nishrao_bucket_1

7. On OCI, create Custom Image from OVA file & create instance (used OCI's VM.Standard2.1 for AWS's t2.micro)


References:
1. https://medium.com/oracledevs/how-to-migrate-or-clone-aws-ec2-vm-into-oracle-oci-cloud-26a13d767eec
2. https://vinception.fr/how-to-migrate-instances-from-aws-to-oracle-cloud/
3. https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/enablingserialconsoleaccess.htm
4. https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport.html
5. https://blogs.oracle.com/linux/post/using-rclone-to-copy-data-in-and-out-of-oracle-cloud-object-storage
6. https://www.ateam-oracle.com/post/migrate-database-backup-from-aws-s3-to-oci-object-storage-using-rclone-and-restore-it-with-help-of-oracle-storage-gateway

** Most useful:
7. https://leandromichelino.medium.com/how-to-migrate-an-amazon-linux-2-ami-to-oci-afc9c75b64bd
3. https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/enablingserialconsoleaccess.htm
4. https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport.html
