---
layout: post
title: Authenticating to GKE without gcloud CLI
excerpt: How to authenticate to GKE without gcloud CLI
tags: kubernetes gcloud
---

How to authenticate to GKE without gcloud CLI

``````
kubectl create serviceaccount k8sadmin -n kube-system

kubectl create clusterrolebinding k8sadmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8sadmin

kubectl create token k8sadmin -n=kube-system

kubectl config set-credentials k8sadmin-token --token=<YOUR-TOKENHERE>

kubectl config set-context gke_elastic-pme-team_us-central1-a_ugo-cluster-arm  --cluster=gke_elastic-pme-team_us-central1-a_ugo-cluster-arm --user=k8sadmin-token

```