# Tracee

[Tracee](https://www.aquasec.com/products/tracee/) is an open source runtime security threat detection engine developed by Aqua Security. Tracy performs runtime security monitoring but uses eBPF instead of working at the application layer like Neuvector does.

Some high-level notes about Tracee (specifically compared to Neuvector):

- Tracee is licensed under the [Apache 2.0 License](https://github.com/aquasecurity/tracee/blob/main/LICENSE)
- Tracee does have upstream [helm charts](https://github.com/aquasecurity/tracee/tree/main/deploy/helm)
- Tracee does NOT come with a UI like Neuvector. You can use other tools like [Falcosidekick]( https://github.com/falcosecurity/falcosidekick) as a message relay. Additionaly (as documented below), you can use the PLG (Promtail, Loki, Grafana) stack for some visualizations.
- Tracee only alerts on rules but does not act on rules. I.e., it is ONLY an intrusion detection system (IDS), not an intrusion prevention system (IPS).
- Tracee does not offer many of the other features that Neuvector does. For example, as mentioned above, it is not an IPS, thefore it cannot be used as a web applicationn firewall.
- Tracee also is not able to do any sort of image security. That is, it cannot perform image whitelisting, nor any runtime or CI/CD image scanning.
- Tracee does allow configuration as code via creating [policies](https://aquasecurity.github.io/tracee/v0.19/docs/policies/) 
- Given the above information, it must be noted that Tracee is much more lightweight than Neuvector.

## Kubernetes quickstart guide

1.) Have a running K3d cluster with DUBBD (Defense Unicorns BigBang Distro) 0.17.0+ and Metallb deployed

2.) Add the following kyverno exceptions (required for Tracee ro run):

```
kyvernoPolicies:
  values:
    policies:
      restrict-host-path-mount:
        exclude:
          any:
            - resources:
                names:
                  - tracee-*
                namespaces:
                  - tracee-system
      restrict-host-path-write:
        exclude:
          any:
            - resources:
                names:
                  - tracee-*
                namespaces:
                  - tracee-system
      restrict-volume-types:
        exclude:
          any:
            - resources:
                namespaces:
                  - tracee-system
                names:
                  - tracee-*
      disallow-host-namespaces:
        exclude:
          any:
            - resources:
                names:
                  - tracee-*
                namespaces:
                  - tracee-system
      disallow-privileged-containers:
        exclude:
          any:
            - resources:
                names:
                  - tracee-*
                namespaces:
                  - tracee-system
```

3.) Additionally, for Tracee's dashboard (covered later) to correctly work in Grafana, the following values overrides are required. THis allows prometheus to gather detection and events metrics from Tracee and display it in Grafana.

```
monitoring:
  values:
    prometheus:
      prometheusSpec:
        additionalScrapeConfigs:
          - job_name: tracee
            scrape_interval: 5s
            metrics_path: /metrics
            static_configs:
              - targets: ['tracee.tracee-system.svc.cluster.local:3366']
```

4.) Create the zarf package
```
cd zarf-package
zarf package create . --confirm 
```

5.) Deploy the zarf package

```
# Assumes you have KUBECONFIG set and can access a kubernetes cluster with the above pre-reqs
zarf package deploy --confirm zarf-package-tracee-amd64.tar.zst 
```

6.) Import Tracee's metrics dashboard:

  i.) Download Tracee's Grafana dashboard file [here](https://github.com/aquasecurity/tracee/blob/main/deploy/grafana/tracee.json). Once the file is downloaded, find all instances of 'w5C9dFs7k' (the prometheus datasource's uid) and replace with 'prometheus'.

  ii.) Go to Grafana's Web UI -> Dashboards -> New -> Import, and Click "Upload dashboard JSON file" and upload the json file. Then click Import. You now have a "Tracee Dashboard" to view in Grafana!


7.) To generate an event, run the following:

```
# Kick off pod that will generate a 'fileless_execution' event
kubectl run tracee-tester --image=docker.io/aquasec/tracee-tester:latest -- TRC-105
# Verify the logs of tracee's daemonset found an event
kubectl logs -f ds/tracee -n tracee-system | grep fileless_execution 
# You should be seeing json entries with '"eventName":"fileless_execution"' in the logs
```

8.) You also should be able to see the metrics now on the Tracee Dashboard in Grafana. Additionally, you can go to (in Grafana) Explore -> select 'Loki' as your datasource and query:
'Label filter': 'app_kubernetes_io_instance" = "tracee" and 'Line contains' = "fileless_execution" to see more details on the events.

## Additional resources

[Overview](https://aquasecurity.github.io/tracee/v0.19/) \
[Github repo](https://github.com/aquasecurity/tracee) \
[Grafana Dashboard](https://aquasecurity.github.io/tracee/latest/tutorials/deploy-grafana-dashboard/) \
[PLG Integration](https://aquasecurity.github.io/tracee/latest/tutorials/promtail/) \
[Policies](https://aquasecurity.github.io/tracee/v0.19/docs/policies/) \
[Events](https://aquasecurity.github.io/tracee/v0.19/docs/events/) \
