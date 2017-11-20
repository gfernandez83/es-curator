# ES Curator
This curator files are mainly used for indices creation, backup and deletion. This was tested on Ubuntu 16.04 using ES 5.5 and Curator version 5.3.0.

## Prerequisites
Ensure you have the following packages:
1. python-pip
```
apt-get python-pip -y
```
2. elasticsearch-curator
```
pip install elasticsearch-curator
```
3. repository-s3
```
/usr/share/elasticsearch/bin/elasticsearch-plugin install repository-s3
```

## Enabling Curator
1. Create a directory for your curator files.
```
mkdir -p /opt/es_curator/.curator
```
2. Copy *.yml files on your .curator directory and change fields accordingly. On your curator.yml file, update log file directory on your desired location.
```
logging:
  loglevel: DEBUG
  logfile: '/opt/es_curator/log/graylog.log'
```
### For Snapshot/Backup
1. Setup your repository, you may use local repo but for this case, I used s3 repository.
#### Seting up S3-repo
Make sure you have the correct IAM role assigned on your system. Has list and write permission on your s3 account, else, it will post an error regarding no permission on master bucket. You won't be able to upload your backup if this happens eventhough your repository was created. Input access key and secret key ID.
```
curl -XPUT http://localhost:9200/_snapshot/s3-backup -d '{
  "type": "s3",
  "settings": {
    "bucket": "es-backup",
    "region": "ap-northeast-1",
    "access_key": "xxxx", 
    "secret_key": "xxxx"
  }
}'
```
Verify repository has been created.
```
curl -XGET 'http://localhost:9200/_snapshot?pretty'
```
2. You may now run test snapshot. Keep in mind that ES does incremental backup by default. --dry-run option is for testing only. Remove it if you want to execute index snapshot.
```
curator ./snapshot.yml --config ./curator.yml --dry-run
```
3. You should now see metadata and indices on your s3 repository. Please do not remove anything from there as it may cause problems on your snapshot. It is suggested to delete snapshot directly from your system. See sample below.
```
curl -XDELETE 'http://localhost:9200/_snapshot/<repo_name>/<snapshot_name>' 
curl -XDELETE 'http://localhost:9200/_snapshot/s3-backup/es-backup-2017-11-10-08:13:43?pretty'
```
4. Your snapshots are useless unless you use it. You may see what indices will be restored from snapshot details.
```
curl -XPOST 'http://localhost:9200/_snapshot/s3-backup/es-backup_2017-11-16/_restore'
```
```
curl -XGET 'http://localhost:9200/_snapshot/s3-backup/_all?pretty'
```
```
{
      "snapshot" : "es-backup_2017-11-20",
      "uuid" : "y2PbHmsvRQivJPLCq-xxxx",
      "version_id" : 5050199,
      "version" : "5.5.1",
      "indices" : [
        "graylog_0",
        "graylog_1"
      ],
      "state" : "SUCCESS",
      "start_time" : "2017-11-20T02:22:03.128Z",
      "start_time_in_millis" : 1511144523128,
      "end_time" : "2017-11-20T02:23:12.430Z",
      "end_time_in_millis" : 1511144592430,
      "duration_in_millis" : 69302,
      "failures" : [ ],
      "shards" : {
        "total" : 8,
        "failed" : 0,
        "successful" : 8
      }
```
5. You may now add the curator to your crontab and how frequent it needs to run. I created a separate bash file to run my snapshot curator but you may also add it directly to your crontab. Just ensure to declare the PATH of you curator to make it work.

curator.sh
```
#!/bin/bash

PATH=/usr/local/bin/curator:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

curator /opt/es_curator/.curator/snapshot.yml --config /opt/es_curator/.curator/curator.yml
```
crontab 
```
0 22 * * * /usr/bin/nice -n19 /usr/bin/ionice -n7 -c2 /opt/es_curator/.curator/curator.sh
```
## Acknowledgement
This was referenced from https://github.com/elastic/curator.
