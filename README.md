# mcd_backup_restore_gitlab
# 1.crontab
0 3 * * * "$(command -v bash)" -c 'pushd /opt/mcd && /bin/bash backup_gitlab.sh && popd'
# 2.backup_gitlab.sh
```python
#!/bin/bash
echo "$(date -Is) ssh v1i.cc \"docker exec -t gitlab gitlab-backup create\"" >> ./sshcommands.log
ssh mcd.v1i.cc "docker exec -t gitlab gitlab-backup create"
/usr/bin/python3 mcd_oss.py uploadbackup >> ./sshcommands.log
#sshfs xx.xx.cc:/data/docker-project/gitlab/gitlab/data/backups /mnt/remote_disk/
```
# 3.mcd_oss.py
```python
# -*- coding: utf-8 -*-
import time

import oss2
import os
import fire
import glob
import traceback
from logger import get_logger
import sys
import time
import collections

logger = get_logger()
endpoint = 'http://oss-cn-beijing.aliyuncs.com'  # Suppose that your bucket is in the Hangzhou region.
backpath = '/mnt/remote_disk/'
auth = oss2.Auth('', '')
bucket = oss2.Bucket(auth, endpoint, '')
service = oss2.Service(auth, 'http://oss-cn-beijing.aliyuncs.com')
thres_hold_total = 90 #90 GB


def removeremotefile(gitlabbackupfilename):
    try:
        os.remove(gitlabbackupfilename)
    except:
        logger.error(f'delete file:{gitlabbackupfilename} error.\n\t{traceback.format_exc()}')
    logger.debug(f'file {gitlabbackupfilename},success deleted.')


def percentage(consumed_bytes, total_bytes):
    if total_bytes:
        rate = int(100 * (float(consumed_bytes) / float(total_bytes)))
        if rate % 10 == 0:
            print('\r{0}% '.format(rate), flush=True)


def uploadbackup():
    while 1:
        if all([xxx.endswith('ee_gitlab_backup.tar') for xxx in glob.glob(os.path.join(backpath, '*'))]):
            break
        else:
            time.sleep(30)
    gitlabbackupfile = '1640679576_2021_12_28_14.5.0-ee_gitlab_backup.tar'
    key = ''
    try:
        for gitlabbackupfile in glob.glob(os.path.join(backpath, '*')):
            while 1:
                if gitlabbackupfile.startswith('-'):
                    logger.debug(f'waiting for dump file {gitlabbackupfile} complete sleep for 30s.')
                    time.sleep(30)
                else:
                    break
            logger.debug(f'+++++++++++++++++++ now we upload:{gitlabbackupfile} begin. +++++++++++++++++++')
            key = gitlabbackupfile.replace(backpath, '')
            bucket.put_object_from_file(key, gitlabbackupfile, progress_callback=percentage)
            logger.debug(f'+++++++++++++++++++ now we upload:{gitlabbackupfile} end. +++++++++++++++++++')
            removeremotefile(gitlabbackupfile)
        for object_info in oss2.ObjectIterator(bucket):
            logger.debug(f'now we have files:{object_info.key}')
    except:
        logger.critical(f'{backpath}-{gitlabbackupfile}-{key} have errors exit.\n\t{traceback.format_exc()}')
        os._exit(0)


def listbackups():
    try:
        for object_info in oss2.ObjectIterator(bucket):
            logger.debug(f'now we have files:{object_info.key}')
    except:
        os._exit(0)


def getbuckettotalsize() -> int:
    total_size = 0
    for object_info in oss2.ObjectIterator(bucket):
        total_size += abs(object_info.size)
    return int(total_size / 1024 ** 3)


def deletthefirstfile():
    tmp_files = {}
    for object_info in oss2.ObjectIterator(bucket):
        if str(object_info.key).endswith('tar'):
            tmp_files[str(object_info.key).split("_")[0]] = str(object_info.key)
    od = collections.OrderedDict(sorted(tmp_files.items()))
    filetobedeleted = od[next(iter(od))]
    bucket.delete_object(filetobedeleted)
    return filetobedeleted


def keepbucketvolum():
    running_count = 0
    logger.debug(f"bucket now total size :{getbuckettotalsize()} GB")
    while (getbuckettotalsize() > thres_hold_total):
        if running_count > 99:
            break
        else:
            running_count += 1
        logger.debug(f"the file :{deletthefirstfile()} is deleted ...")
        logger.debug(f"after delete bucket now total size :{getbuckettotalsize()} GB")
        time.sleep(1)


if __name__ == '__main__':
    logger.debug(f'files tobe uploaded to oss:\n\t{glob.glob(os.path.join(backpath, "*"))}')
    keepbucketvolum()
    fire.Fire()

````
# 4.restore.sh
``` bash
# Stop the processes that are connected to the database
docker exec -it gitlab gitlab-ctl stop puma
docker exec -it gitlab gitlab-ctl stop sidekiq

# Verify that the processes are all down before continuing
docker exec -it gitlab gitlab-ctl status

# Run the restore. NOTE: "_gitlab_backup.tar" is omitted from the name
docker exec -it gitlab gitlab-backup restore BACKUP=1640762859_2021_12_29_14.5.0-ee

# Restart the GitLab container
docker restart gitlab

# Check GitLab
docker exec -it gitlab gitlab-rake gitlab:check SANITIZE=true

#warning: Your gitlab.rb and gitlab-secrets.json files contain sensitive data
#and are not included in this backup. You will need to restore these files manually.
#Restore task is done.
```

