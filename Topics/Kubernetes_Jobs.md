## KUBERNETES JOBS

- Create a job to calculate pi using the perl image
```sh
kubectl create job calculatepi --image=perl:5.34.0 -- "perl" "-Mbignum=bpi" "-wle" "print bpi(2000)"
```

- Watch the job completion, can take up to 60 seconds
```sh
watch kubectl get jobs
```

- If we run a describe against this job, we can see that this completed
```sh
kubectl describe job/calculatepi

k get pods -o wide
```

- Lets view the logs from the pod, showing pi to 2000
```bash
k logs <POD_NAME>
```

- We'll capture the yaml from our pi job to a file called calculatepi. Read about `completions` and `parallelism`.
```bash
kubectl create job calculatepi --image=perl:5.34.0 --dry-run=client -o yaml -- "perl" "-Mbignum=bpi" "-wle" "print bpi(2000)" | tee calculatepi.yaml

kubectl explain job.spec | more
```

- Update the calculatepi.yaml to include completions of 20 and parallelism of 5 
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: calculatepi
spec:
  completions: 20
  parallelism: 5
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - perl
        - -Mbignum=bpi
        - -wle
        - print bpi(2000)
        image: perl:5.34.0
        name: calculatepi
        resources: {}
      restartPolicy: Never
status: {}
```
```bash
k apply -f calculatepi.yaml && sleep 1 && watch kubectl get pods -o wide
```


- check logs of any pods (20 Pods) and you will see the vaue of PI calculated
```bash
k logs <POD_NAME>

kubectl delete job/calculatepi
```

## KUBERNETES CronJobs

- We'll re-use the previous example but we will repurpose it as a CronJob with a schedule of * * * * *
```bash
kubectl create cronjob calculatepi --image=perl:5.34.0 --schedule="* * * * *" -- "perl" "-Mbignum=bpi" "-wle" "print bpi(2000)"

k get cronjob
```

- If we watch this, as the minute ticks over it should start, you may need to toggle the tutorial button to close this pane to see this. 
```bash
watch kubectl get jobs
```
_FYI: Regardless of how many jobs are completed, we will only have 3 running in total, it may briefly step up to 4 but then it will go back to 3. This is because of the `successfulJobsHistoryLimit` parameter._

-  You can edit this file and change it to your preference. Add in `completions: 20` and `parallelism: 5` under `spec.jobTemplate`.spec, save the file and exit when done
```bash
kubectl edit cronjob/calculatepi

watch kubectl get jobs
```

_FYI: The parameter completions: 20 signify that the job will create 20 pods overall to do the task_
_FYI: The parameter parallelism: 5 signify in a Kubernetes Job specification that maximum number of pods that should be running in parallel_

- Cleanup
```bash
kubectl delete cronjob/calculatepi

kubectl get jobs

kubectl get pods
```

_FYI: How many completed and failed jobs are kept by default according to the `successfulJobsHistoryLimit` field in a CronJob, the answer is `3`_

