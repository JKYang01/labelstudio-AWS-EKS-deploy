## Update labelstudio using helm

Modify the Helm Chart Values
If you installed Label Studio via Helm, you'll want to add the SSRF_PROTECTION_ENABLED environment variable to the Helm values. You can do this by either updating your existing values.yaml file or using the --set command when upgrading the Helm release.

Option 1: Using values.yaml
Add the following under the environment variables section of your values.yaml:


```global:
  extraEnvironmentVars:
    SSRF_PROTECTION_ENABLED: "true"
```

Then, upgrade your Helm release:
`$ helm upgrade labelstudio your-chart-repo/labelstudio -f values.yaml --namespace default `

Option 2: Using --set Command
If you prefer not to edit the values.yaml directly, you can set the environment variable via the --set flag:

```
helm upgrade labelstudio your-chart-repo/labelstudio \
  --set global.extraEnvironmentVars.SSRF_PROTECTION_ENABLED="true" \
  --namespace default
```

2. Verify the Deployment
After applying the change, ensure that your pods are updated with the new environment variable:
`$ kubectl describe pod <labelstudio-pod-name> -n default | grep SSRF_PROTECTION_ENABLED`
