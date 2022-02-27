## Rados Object Storage Gateway

Demo video on YouTube [here](https://youtu.be/lmFdpLipaBA) \

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

4. Now you have a choice, you can manually create a Pool if you want to use erasure coding data protection, or you can skip this step and Ceph will create it for you as a 'replica'. I chose to create an erasure coded pool with a format of 4x2. The naming convention is 'zone' ie 'default' for me, 'rgw', then 'buckets' and then 'data' : `default.rgw.buckets.data`.

5. Create the erasure coded profile
    - ceph osd erasure-code-profile set ec-4x2-default k=4 m=2
    - ceph osd pool create ecpool erasure ec-4x2-default

6. Create the default erasure coded pool (step 4)
    - ceph osd pool create default.rgw.buckets.data erasure ec-4x2-default 

7. Assign the pool to rgw
    - ceph osd pool application enable default.rgw.buckets.data rgw

8. Create our user
    - radosgw-admin user create --uid=$user --display-name="$user"
        ##### Copy the access_key and secret_key, you'll need them later, if you forget, you can retrieve them with `radosgw-admin user info --uid=$user`

9. If you want to create a Swift sub user, this is how.
    - radosgw-admin key create --subuser=$user:swift --key-type=swift --gen-secret

10. You have another choice to make, you can either create your first bucket in the UI like I did (recommended) or you can use the s3cmd command. We'll use the s3cmd next, but for now how about you follow along in the video and create your first bucket using the UI.

11. If you haven't done it already, install s3cmd... `dnf install s3cmd -y`

12. Let's configure your s3cmd profile... please note, I have created a wildcard DNS record that points to my HAPROXY that resolves *.s3.redcloud.land.
    - s3cmd --configure

13. S3CMD Options
    - Access Key: $USER_ACCESS_KEY (from step 8)
    - Secret Key: $USER_SECRET_KEY (from step 8)
    - S3 Endpoint: s3.redcloud.land
    - DNS Style Endpoint: s3.redcloud.land
    - I skipped encrytion
    - Use HTTPS: Yes (The HAPROXY is holding the certificate for port: 443 (https))
    - Test Credentials: No
    - Save Configuration: Yes

14. Modify the S3CMD profile
    - I copied my CA to /etc/ceph/ca.crt
    - vi .s3cfg
        - ca_certs_file = /etc/ceph/ca.crt

15. Let's test access!
    - s3cmd ls 
        ```
        2022-02-26 14:48  s3://$bucket
        ```
    - dd if=/dev/zero of=myfile bs=1024 count=1024000
    - s3cmd put myfile s3://$bucket
    - s3cmd rm s3://$bucket/myfile

HUZZAH! You have a working S3/Object Storage Gateway on Ceph!


### Grafana Dashboard bonus!
Grafana runs on port :3000, for me it's only on my cephadm host (172.16.1.39)

1. Edit the ceph.conf to allow Grafana integration
    - vi /etc/ceph/ceph.conf
       ```
       [security]
       allow_embedding = true
       ```
2. Since I am using private and self-signed certs, we should first disable ssl validation checks
    - ceph dashboard set-grafana-api-ssl-verify False
    - ceph dashboard set-alertmanager-api-ssl-verify False
    - ceph dashboard set-prometheus-api-ssl-verify False

3. Next, let's log into Grafana to and fix the browser error.
    - https://172.16.1.39:3000
        ##### mouse click anywhere on the page and type "thisisunsafe"
    - Now, go back to your Ceph dashboard and check for the performance graphs to be loaded.\
    ***Let me know if this doesn't work for you.***

    <img src= "https://github.com/ocpdude/ceph5-install/blob/main/rados/grafana.png" alt="Grafana in Ceph" width="640" height="405">
 
