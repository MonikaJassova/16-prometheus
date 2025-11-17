# Monitoring with Prometheus

This repo showcases
- installing Prometheus Stack in Kubernetes
- configuring alerting
- configuring monitoring for a 3rd-party application
- configuring monitoring for an own application

## Technologies used

Prometheus, Kubernetes, Helm, AWS EKS, eksctl, Grafana, Redis, Node.js, Docker, DockerHub, Linux

## Installing Prometheus Stack in Kubernetes

1. Created an Amazon EKS cluster on AWS using eksctl: `eksctl create cluster`
1. Deployed microservices app to the cluster: `kubectl apply -f microservices-config.yaml`
1. Deployed Prometheus stack using Prometheus Operator Helm chart
    - added the Helm repo: `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
    - updated the index: `helm repo update`
    - created a separate namespace: `kubectl create ns monitoring`
    - installed the Helm chart: `helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring`
    - verified the pods are running: `kubectl -n monitoring get pods -l "release=monitoring"`

## Configuring Alerting

1. Created [alert-rules.yaml](./alert-rules.yaml) as a PrometheusRule CRD with 2 custom rules
    - when CPU usage exceeds 50 % on a Kubernetes node
    - when a Pod cannot start (in a crash loop)
1. Deployed the custom Kubernetes resource for Prometheus Operator to pick up and orchestrate adding it to the rule file and reloading Prometheus Server: `kubectl apply -f alert-rules.yaml`
    - verified with `kubectl get PrometheusRule -n monitoring` and viewing Alerts in Prometheus Web UI (port forwarded to localhost with `kubectl port-forward service/monitoring-kube-promethues-prometheus -n monitoring 9090:9090 &`)
    - debugging logs with `kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader` and `kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c prometheus`
