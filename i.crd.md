# Extend the Kubernetes API with CRD (CustomResourceDefinition)

- Note: CRD is part of the new CKAD syllabus. Here are a few examples of installing custom resource into the Kubernetes API by creating a CRD.

kubectl api-resources ; look into what this does
kubectl api-versions ; lists current valid apis

Fine. k8s docs, copy and modify the sample crd

## CRD in K8s

### Create a CustomResourceDefinition manifest file for an Operator with the following specifications :
* *Name* : `operators.stable.example.com`
* *Group* : `stable.example.com`
* *Schema*: `<email: string><name: string><age: integer>`
* *Scope*: `Namespaced`
* *Names*: `<plural: operators><singular: operator><shortNames: op>`
* *Kind*: `Operator`

34.yaml















<details><summary>show</summary>
<p>

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: operators.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                email:
                  type: string
                name:
                  type: string
                age:
                  type: integer
  scope: Namespaced
  names:
    plural: operators
    singular: operator
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: Operator
    shortNames:
    - op
```

</p>
</details>

### Create the CRD resource in the K8S API

Misleading request, but it just means to apply the file

<details><summary>show</summary>
<p>

```bash
kubectl apply -f operator-crd.yml
```

</p>
</details>

### Create custom object from the CRD

Didn't know that the api version is .spec.group/.spec.group.versions.name
I could have extrapolated this from the docs

35.yaml

* *Name* : `operator-sample`
* *Kind*: `Operator`
* Spec:
  * email: `operator-sample@stable.example.com`
  * name: `operator sample`
  * age: `30`

<details><summary>show</summary>
<p>

```yaml
apiVersion: stable.example.com/v1
kind: Operator
metadata:
  name: operator-sample
spec:
  email: operator-sample@stable.example.com
  name: "operator sample"
  age: 30
```

```bash
kubectl apply -f operator.yml
```

</p>
</details>

### Listing operator

<details><summary>show</summary>
<p>

Use singular, plural and short forms

```bash
kubectl get operators

kubectl get operator

kubectl get op
```

</p>
</details>

If the CKAD questions are a simple as the above, then great! The real challenge will lie in coming up with a legitimate use case for my own crd. 'Chaos monkey' might be that use case