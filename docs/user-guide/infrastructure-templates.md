# Infrastructure Templates

Templates are text files which describe infrastructure to be deployed. Before proceeding we recommend you read through the [Brent Resource Package structure](http://servicelifecyclemanager.com/2.1.0/user-guides/resource-engineering/resource-packages/brent/create-brent-resource-package/) section of the Lifecycle Manager user guide to understand where the infrastructure template fits in the Resource structure. 

Currently the Kubernetes VIM driver supports two template types:

- `ObjectConfiguration` - based on the [Kubernetes object configuration file format](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- `Helm` - describes an installation of a Helm chart

## Object Configuration

An object configuration file is written in YAML and may include multiple objects separated by the YAML document separator:

```
#Example Object Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-a
data:
  dataA: valueA
  dataB: valueB
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-b
data:
  dataA: valueA
  dataB: valueB
  dataC: valueC
```

The file may include any object type understood by your target Kubernetes cluster, including [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) you have deployed. 

A simpe rule of thumb: if you can create it with `kubectl apply` then you should be able to create it with this driver.

### Templating Object Configuration

The object configuration file may make use of [Jinja2 template variables syntax](https://jinja.palletsprojects.com/en/2.10.x/templates/#variables) to inject property values from the Resource descriptor:

```
## Example template syntax to inject value of property "dataA"
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-a
data:
  dataA: {{ dataA }}
  dataB: valueB
```

The template is rendered as text before parsed as YAML, so there is no need to wrap the template syntax in quotes when it's the first item on a line.

You may also use [control structures](https://jinja.palletsprojects.com/en/2.11.x/templates/#if) to configure optional elements of the object:

```
## Example template syntax for conditional block based on the "protocol" property value
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-a
data:
{% if protocol == 'https' -%}
  port: 443
  ssl: true
{% elif protocol == 'http' -%}
  port: 80
  ssl: false
{% else -%}
  customProtocol: {{ protocol }}
  ssl: false
{-% endif %}
```

For full details on the properties which may be used in templates, read the [Property Handling](./property-handling.md) section of this user guide.

### Configuring your Resource

To use the object configuration file as the infrastructure template in your Resource the descriptor must specify the file and type under `infrastructure`:

```
...descriptor omitted...

infrastructure:
  Kubernetes:
    template:
      file: <name of configuration file template>
      template-type: ObjectConfiguration

...descriptor omitted...
```

## Helm Charts

To use Helm charts instead of object configuration files, you need to create a template file which describes the chart and values to use on installation (we'll call this the helm deployment file and it is written in YAML). The chart must be declared as a base64 encoded string copy of your chart tgz, whilst the values must be a multiline string template for your values file.

To base64 encode your chart on the command line, use `base64`:

```
base64 <path to chart>.tgz
```

This will output the string you need to copy into your helm deployment file. Make sure you indent all lines by 2 spaces.

Create a helm deployment file in your Resource infrastructure directory with the encoded chart string and include your values file content:

```
chart: |
  <base64 encoded chart>
values: |
  valueA: someValue
  sub-chart:
    value-in-subchart: someValue
```

The values section may make use of [Jinja2 template variables syntax](https://jinja.palletsprojects.com/en/2.10.x/templates/#variables) to inject property values from the Resource descriptor:

```
chart: |
  <base64 encoded chart>
values: |
  valueA: {{ dataA }}
  sub-chart:
    value-in-subchart: someValue
```

You may also use [control structures](https://jinja.palletsprojects.com/en/2.11.x/templates/#if) to configure optional elements of the values file:

```
chart: |
  <base64 encoded chart>
values: |
{% if protocol == 'https' -%}
  port: 443
  ssl: true
{% elif protocol == 'http' -%}
  port: 80
  ssl: false
{% else -%}
  customProtocol: {{ protocol }}
  ssl: false
{-% endif %}
```

For full details on the properties which may be used in templates, read the [Property Handling](./property-handling.md) section of this user guide.

### Configuring your Resource

To use the helm charts as the infrastructure template in your Resource the descriptor must specify the helm deployment file and type under `infrastructure`:

```
...descriptor omitted...

infrastructure:
  Kubernetes:
    template:
      file: <name of helm deployment file>
      template-type: Helm

...descriptor omitted...
```

In addition, you may control the namespace the chart is deployed to by adding a namespace property to the descriptor (otherwise the default on the deployment location is used):

```
properties:
  namespace:
    type: string
```