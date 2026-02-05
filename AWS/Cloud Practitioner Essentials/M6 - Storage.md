### **Recap and next steps**

In this module, you learned about the diverse storage options available in AWS, starting with block storage services like Amazon EC2 Instance Store and Amazon EBS. You learned how Amazon EBS provides persistent block storage volumes for EC2 instances, while EC2 instance store offers temporary block-level storage. You learned how to use EBS snapshots and AWS Data Lifecycle Manager for automated backup management and data protection.

You then examined Amazon S3, a highly scalable object storage service that serves as a foundation for many cloud storage needs. You delved into file storage solutions, including Amazon Elastic File System (Amazon EFS) for Linux-based workloads and Amazon FSx for Windows, Lustre, OpenZFS, and NetAPP ONTAP file systems. Finally, you learned about AWS Storage Gateway, which bridges on-premises environments with AWS storage services to enable hybrid cloud storage architectures.

### Resources

To learn more about the material covered in this module, choose the resource links contained in the following table.

|Resource link|Description|
|---|---|
|[Amazon EC2 Instance Store User Guide(opens in a new tab)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)|A temporary storage option that is directly attached to the host computer of an EC2 instance, providing high-performance but non-persistent storage.|
|[Amazon Elastic Block Store (Amazon EBS)(opens in a new tab)](https://aws.amazon.com/ebs/)|A scalable block storage service that provides persistent, high-performance volumes you can attach to your EC2 instances for data storage and applications.|
|[Amazon Elastic Block Store (Amazon EBS) FAQ(opens in a new tab)](https://aws.amazon.com/ebs/faqs/)|Frequently asked questions about Amazon EBS.|
|[Amazon EBS Snapshots User Guide(opens in a new tab)](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)|EBS Snapshots are point-in-time backups of your cloud storage volumes, making it possible to protect data and restore it when needed.|
|[Amazon Data Lifecycle Manager User Guide(opens in a new tab)](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html)|A service that streamlines the creation, retention, and deletion of Amazon EBS snapshots.|
|[Amazon Simple Storage Service (Amazon S3)(opens in a new tab)](https://aws.amazon.com/s3/)|A scalable cloud storage service that can store and retrieve any amount of data from anywhere on the web.|
|[Amazon Simple Storage Service (Amazon S3) FAQ(opens in a new tab)](https://aws.amazon.com/s3/faqs/)|Frequently asked questions about Amazon S3.|
|[Amazon S3 Storage Classes(opens in a new tab)](https://aws.amazon.com/s3/storage-classes/)|Amazon S3 offers various storage classes, from high-performance frequent access to cost-effective archival options, tailored to different data retrieval needs and budget constraints.|
|[Amazon S3 Versioning User Guide(opens in a new tab)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)|Amazon S3 versioning keeps multiple variants of objects, offering recovery from unintended deletions or modifications by preserving every update to your files.|
|[Amazon S3 Buckets User Guide(opens in a new tab)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-buckets-s3.html)|S3 buckets are cloud storage containers that securely hold various types of data, allowing convenient access and management through the AWS online infrastructure.|
|[Amazon Elastic File System (Amazon EFS)(opens in a new tab)](https://aws.amazon.com/efs/)|A scalable, fully-managed file storage service that lets multiple AWS resources access shared data simultaneously without capacity planning.|
|[Amazon Elastic File System (Amazon EFS) FAQ(opens in a new tab)](https://aws.amazon.com/efs/faq/)|Frequently asked questions about Amazon EFS.|
|[Amazon FSx(opens in a new tab)](https://aws.amazon.com/fsx/)|A fully managed file storage service that lets you launch and run file systems like Windows File Server, Lustre, NetApp ONTAP, and OpenZFS in the AWS cloud.|
|[Amazon FSx for Windows File Server(opens in a new tab)](https://aws.amazon.com/fsx/windows/)|An Amazon FSx option providing reliable, high-performance file storage compatible with Windows applications in the AWS Cloud.|
|[Amazon FSx for NetApp ONTAP(opens in a new tab)](https://aws.amazon.com/fsx/netapp-ontap/)|An Amazon FSx option providing file storage with advanced data management capabilities and compatibility with both Windows and Linux workloads on AWS.|
|[Amazon FSx for OpenZFS(opens in a new tab)](https://aws.amazon.com/fsx/openzfs/)|An Amazon FSx option that provides high-performance, scalable storage using the popular open-source ZFS file system.|
|[Amazon FSx for Lustre(opens in a new tab)](https://aws.amazon.com/fsx/lustre/)|An Amazon FSx option designed to accelerate workloads by providing fast data access for compute-intensive applications in AWS.|
|[AWS Storage Gateway(opens in a new tab)](https://aws.amazon.com/storagegateway)|A hybrid cloud storage service that provides seamless and secure integration between on-premises environment and AWS cloud storage services.|
|[Amazon S3 File Gateway(opens in a new tab)](https://aws.amazon.com/storagegateway/file/s3/)|A Storage Gateway configuration that provides local file access to S3 objects while caching frequently accessed data locally for faster retrieval.|
|[Tape Gateway(opens in a new tab)](https://aws.amazon.com/storagegateway/vtl/)|A Storage Gateway configuration used for backing up data to Amazon S3 while maintaining compatibility with existing tape-based backup applications.|
|[Volume Gateway(opens in a new tab)](https://aws.amazon.com/storagegateway/volume/)|A Storage Gateway configuration that provides iSCSI block storage volumes to on-premises applications, offering both cached and stored modes.|