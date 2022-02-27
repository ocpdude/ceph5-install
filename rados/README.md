## Rados Object Storage Gateway

Demo video on YouTube [here](https://youtu.be/lmFdpLipaBA)

This script assumes you've got a clean Ceph 5 install. I touch on the fact that my lab has a private CA for generating certificates, I use one of these certificates for RGW/SSL connections on HAPROXY. You can easily create self signed certificates if for testing, generating these certs are out of scope for this write up. If you move forward without SSL, set your HAPROXY accordingly. 

For my testing, I use 's3cmd' which is supported as a package for many Linux distro's, you can also find it [here](https://s3tools.org/s3cmd).

1. Label your hosts with _rgw_ or select them manually with you deploy the service. 
    - ceph orch host label add ceph0 rgw
    - ceph orch host label add ceph1 rgw

2. Deploy the rgw service
    - ceph orch apply rgw '--placement=label:rgw' default --ssl=false --port-8000

3. **Checks**
    - Verify the user 'dashboard' was created
    - Verify your rgw services started 'ceph orch ls'
    - Verify you now have access to the Object Gateway UI
    <img src= "https://github.com/ocpdude/ceph5-install/blob/main/rados/dash-rgw.png" alt="Dashboard with Object Gateway" width="640" height="270">

4. 
