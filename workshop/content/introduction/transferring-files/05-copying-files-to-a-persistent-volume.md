If you are mounting a persistent volume into the container for your application and you need to copy files into it, then ``oc rsync`` can be used in the same way as described previously to upload files. All you need to do is supply as the target directory, the path of where the persistent volume is mounted in the container.

If you haven't as yet deployed your application, but are wanting to prepare in advance a persistent volume with all the data it needs to contain, you can still claim a persistent volume and upload the data to it. In order to do this, you will though need to deploy a dummy application against which the persistent volume can be mounted.

To create a dummy application for this purpose run the command:

```execute
oc run dummy --image centos/httpd-24-centos7
```

We use the ``oc run`` command as it creates just a deployment configuration and managed pod. A service is not created as we don't actually need the application we are running here, an instance of the Apache HTTPD server in this case, to actually be contactable. We are using the Apache HTTPD server purely as a means of keeping the pod running.

To monitor the startup of the pod and ensure it is deployed, run:

```execute
oc rollout status dc/dummy
```

Once it is running, you can see the more limited set of resources created, as compared to what would be created when using ``oc new-app``, by running:

```execute
oc get all --selector run=dummy -o name
```

Now that we have a running application, we next need to claim a persistent volume and mount it against our dummy application. When doing this we assign it a claim name of ``data`` so we can refer to the claim by a set name later on. We mount the persistent volume at ``/mnt`` inside of the container, the traditional directory used in Linux systems for temporarily mounting a volume.

```execute
oc set volume dc/dummy --add --name=tmp-mount --claim-name=data --type pvc --claim-size=1G --mount-path /mnt
```

This will cause a new deployment of our dummy application, this time with the persistent volume mounted. Again monitor the progress of the deployment so we know when it is complete, by running:

```execute
oc rollout status dc/dummy
```

To confirm that the persistent volume claim was successful, you can run:

```execute
oc get pvc
```

With the dummy application now running, and with the persistent volume mounted, capture the name of the pod for the running application.

```execute
POD=`pod run=dummy`; echo $POD
```

We can now copy any files into the persistent volume, using the ``/mnt`` directory where we mounted the persistent volume, as the target directory. In this case since we are doing a one off copy, we can use the ``tar`` strategy instead of the ``rsync`` strategy.

```execute
oc rsync ./ $POD:/mnt --strategy=tar
```

When complete, you can validate that the files were transferred by listing the contents of the target directory inside of the container.

```execute
oc rsh $POD ls -las /mnt
```

If you were done with this persistent volume and perhaps needed to repeat the process with another persistent volume and with different data, you can unmount the persistent volume but retain the dummy application.

```execute
oc set volume dc/dummy --remove --name=tmp-mount
```

Monitor the process once again to confirm the re-deployment has completed.

```execute
oc rollout status dc/dummy
```

Capture the name of the current pod again:

```execute
POD=`pod run=dummy`; echo $POD
```

and look again at what is in the target directory. It should be empty at this point. This is because the persistent volume is no longer mounted and you are looking at the directory within the local container file system.

```execute
oc rsh $POD ls -las /mnt
```

If you already have an existing persistent volume claim, as we now do, you could mount the existing claimed volume against the dummy application instead. This is different to above where we both claimed a new persistent volume and mounted it to the application at the same time.

```execute
oc set volume dc/dummy --add --name=tmp-mount --claim-name=data --mount-path /mnt
```

Look for completion of the re-deployment:

```execute
oc rollout status dc/dummy
```

Capture the name of the pod:

```execute
POD=`pod run=dummy`; echo $POD
```

and check the contents of the target directory. The files we copied to the persistent volume should again be visible.

``oc rsh $POD ls -las /mnt
```

When done and you want to delete the dummy application, use ``oc delete`` to delete it, using a label selector of ``run=dummy`` to ensure we only delete the resource objects related to the dummy application.

```execute
oc delete all --selector run=dummy
```

Check that all the resource objects have been deleted.

```execute
oc get all --selector run=dummy -o name
```

Although we have deleted the dummy application, the persistent volume claim still exists and can later be mounted against your actual application to which the data belongs.

```execute
oc get pvc
```
