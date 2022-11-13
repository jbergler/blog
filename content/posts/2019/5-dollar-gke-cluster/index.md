+++
date = "2019-05-05T20:41:11Z"
title = "$5 GKE cluster"
+++

Over the years my personal hosting story has changed many times.
My first real job was at a small ISP at a time when the average internet connection at home clocked in at a whopping 512kbps. They had some empty rackspace in their server room and I jumped at the chance to rack an old tower.

That was great until I switched jobs and moved countries a few years later.
I was still friends with my old co-workers and they let me keep the machine racked up.
It was hard to fix things when I did something stupid and it didn't take long before I got fed up and started paying for a VPS somewhere.

Managing my own server has always been a great way to learn new things. I learnt rsync and inotify to sync files between computers before Dropbox. I learnt about SIP and how to send SMS using a 3310 connected to a serial port while running my own phone system before Twilio.

I've been running a 3 node kubernetes cluster on DigitalOcean for the last 12 months, but this was before they provided a managed service. I found myself spending more time trying to remember how I set the damn thing up, than I did actually using the cluster. On top of that, my efforts to make nginx-ingress resilient only lead to frustration with various failure modes when one of my nodes ran out of memory (often my master).

I wanted to learn more about kubernetes, but I didn't want to do the operational work to keep the thing running. So I set out to rebuild things from scratch with that in mind.

# Planning my cluster
I have a few requirements:

* Should be a managed kubernetes service, I don't want to run my own.
* Should support autoscaling, but most of the time 1 node should be sufficient.
* I'm willing to tolerate up to 60 seconds of downtime per day, ie [99.95%](https://en.wikipedia.org/wiki/High_availability#Percentage_calculation)
* I'm willing to build some tooling myself, as long as it doesn't need regular attention once its running.
* I'm hoping to spend less than $10/month.

Almost every big cloud player offers a hosted kubernetes solution these days, so lets look at the options.

* Google's GKE doesn't charge a fee for the control plane, but the minimum cost for a load balancer is around $18/month.
* Amazon's EKS charges $146/month ($0.20/hour) for the control plane and ~$14/month ($0.02/hour) for a load balancer bringing the minimum cost to well over $150/month before we add any nodes.
* Microsoft's AKS doesn't charge a fee for the control and provides their basic load balancers for free.
* DigitalOceans Kubernetes doesn't charge a fee for the control plane, and each load balancer is ~$11/month ($0.015/hour)

At this stage, I'm ruling out EKS since it's just too expensive for a personal project like this.

My next step is to figure out what size nodes I need. While I couldn't find any explicit requirements, the internet seems to be in general consensus that a non-master node should have at least 0.5 vCPU's, 1GB of memory and 10GB of storage.

Google Cloud: g1-small (0.5 vCPU, 1.7GB RAM) ~$5.50/month (preemptible)
Microsoft Azure: B1S (1 vCPU, 1GB RAM) ~$10.50/month
DigitalOcean: standard (1 vCPU, 2GB RAM) ~$10/month (smaller nodes and autoscaling are not supported)

I decided to choose google. It looks the cheapest and I've been meaning to explore and learn more about GCP regardless.

# Setting up GKE

## Installing the gcloud CLI.
After following [these](https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu) instructions, I had the `gcloud` cli tool and was logged in and almost ready to go. The last thing I needed to do what choose my project and region.

```
gcloud config set project "jonasbergler-com"
gcloud config set compute/region "us-west1"
gcloud config set compute/zone "us-west1-c"
```

## Creating the cluster
Since I want to be able to build and destroy GKE clusters as and when I please, I'll need unique names for them.
Let's build names based on the EFF short word list.

```
cluster_name=$(curl -s https://www.eff.org/files/2016/09/08/eff_short_wordlist_1.txt | shuf -n 2 | awk '{printf (NR%2!=0) ? $2 "-" : $2 "\n" }')
echo "Cluster name: ${cluster_name}"
```

Next, let's create a GKE cluster. We're going to run 1.12 and provision 1 preemptible g1-small instance to start.
```
gcloud container clusters create "${cluster_name}" \
  --num-nodes 1 \
  --cluster-version=1.12.7-gke.10 \
  --enable-autoscaling --min-nodes 1 --max-nodes 3 \
  --machine-type g1-small --disk-size=10G \
  --preemptible \
  --tags "gke-${cluster_name}-node" \
  --enable-ip-alias \
  --addons=""
```
> `--addons=""` disables http load balancing (since I don't want to pay for load balancers)

After a few minutes we should have a working cluster and we can configure kubectl.
```
gcloud container clusters get-credentials "${cluster_name}"
```
> We can check this is working with `kubectl top node`

Lastly, since role based access control (RBAC) is enabled by default, we need to grant ourselves the `cluster-admin` role.
```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user $(gcloud config get-value account)
```

# Living without a load-balancer
To make our cluster work without a load balancer that costs money, we will install the nginx.org ingress as a DaemonSet (which ensures it runs on every node) and use DNS to round robin traffic across nodes in our cluster.

## Installing nginx-ingress
```
kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/deployments/common/ns-and-sa.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/deployments/common/nginx-config.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/deployments/common/default-server-secret.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/deployments/rbac/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/deployments/daemon-set/nginx-ingress.yaml
```

## Firewall rules
To ensure that traffic actually makes it to the nginx-ingress we must create some firewall rules.
These leverage the tag we specified when we created the cluster to target the right nodes.

```
gcloud compute firewall-rules create "allow-http" --allow tcp:80 --source-ranges "0.0.0.0/0" --target-tags "gke-${cluster_name}-node"
gcloud compute firewall-rules create "allow-https" --allow tcp:443 --source-ranges "0.0.0.0/0" --target-tags "gke-${cluster_name}-node"
```

## Keeping DNS up to date
This is where things get a little trickier. Since we are using pre-emptible nodes to keep the cost down, we expect instances to be rotated on a regular basis and will need to ensure DNS stays up to date.

To solve this problem, I created [node-ip-controller](https://github.com/jbergler/node-ip-controller/). It is a small Kubernetes Controller which watches Node resources in a cluster and updates a record in Cloud DNS when things change.

# Update: Dec 2019
In the 6 months after I set up this cluster things went well. The average cost was just over $5/month and I was able to tinker with kubernetes without the hassle of running the control plane.

{{< image src="graph.png" >}}

{{< image src="table.png" >}}

The only public facing thing I've run is this blog, which I was previously hosting for free on gitlab pages. But I've also been running a few small tools like influx + grafana for my various projects.
Recently, I've scaled the cluster up to handle more load and will be looking at the new e2 instance types which promise slightly more resources for less money.
