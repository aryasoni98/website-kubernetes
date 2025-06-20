---
reviewers:
- thockin
- dwinship
min-kubernetes-server-version: v1.29
title: सर्विस आईपी रेंज का विस्तार करें
content_type: task
---

<!-- overview -->
{{< feature-state feature_gate_name="MultiCIDRServiceAllocator" >}}

यह दस्तावेज़ बताता है कि क्लस्टर को असाइन की गई मौजूदा सर्विस आईपी रेंज का विस्तार कैसे करें।


## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}}

{{< version-check >}}

{{< note >}}
हालांकि आप इस सुविधा का उपयोग पुराने संस्करण के साथ कर सकते हैं, यह सुविधा केवल v1.33 से GA और आधिकारिक तौर पर समर्थित है।
{{< /note >}}

<!-- steps -->

## सर्विस आईपी रेंज का विस्तार करें

कुबेरनेट्स क्लस्टर जिनमें kube-apiservers ने `MultiCIDRServiceAllocator`
[फीचर गेट](/docs/reference/command-line-tools-reference/feature-gates/) सक्षम किया है और
`networking.k8s.io/v1beta1` API समूह सक्रिय है, एक ServiceCIDR ऑब्जेक्ट बनाएंगे जो
`kubernetes` का प्रसिद्ध नाम लेता है, और जो kube-apiserver को `--service-cluster-ip-range` कमांड लाइन तर्क के मान के आधार पर एक आईपी पता रेंज निर्दिष्ट करता है।

```sh
kubectl get servicecidr
```

```
NAME         CIDRS          AGE
kubernetes   10.96.0.0/28   17d
```

प्रसिद्ध `kubernetes` सर्विस, जो पॉड्स को kube-apiserver एंडपॉइंट को उजागर करती है, डिफ़ॉल्ट ServiceCIDR रेंज से पहले आईपी पते की गणना करती है और उस आईपी पते को अपने
क्लस्टर आईपी पते के रूप में उपयोग करती है।

```sh
kubectl get service kubernetes
```

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   17d
```

डिफ़ॉल्ट सर्विस, इस मामले में, ClusterIP 10.96.0.1 का उपयोग करती है, जिसमें संबंधित IPAddress ऑब्जेक्ट होता है।

```sh
kubectl get ipaddress 10.96.0.1
```

```
NAME        PARENTREF
10.96.0.1   services/default/kubernetes
```

ServiceCIDRs को {{<glossary_tooltip text="फाइनलाइज़र" term_id="finalizer">}} से सुरक्षित किया जाता है,
ताकि सर्विस ClusterIPs अनाथ न रह जाएं; फाइनलाइज़र केवल तभी हटाया जाता है जब कोई अन्य सबनेट
हो जिसमें मौजूदा IPAddresses हों या सबनेट से संबंधित कोई IPAddresses न हों।

## सेवाओं के लिए उपलब्ध आईपी की संख्या बढ़ाएँ

ऐसे मामले हैं जब उपयोगकर्ताओं को सेवाओं के लिए उपलब्ध पतों की संख्या बढ़ाने की आवश्यकता होगी,
पहले, सर्विस रेंज को बढ़ाना एक विघटनकारी ऑपरेशन था जो डेटा हानि का कारण भी बन सकता था।
इस नई सुविधा के साथ उपयोगकर्ताओं को केवल उपलब्ध पतों की संख्या बढ़ाने के लिए एक नया ServiceCIDR जोड़ना होगा।

### एक नया ServiceCIDR जोड़ना

सेवाओं के लिए 10.96.0.0/28 रेंज वाले क्लस्टर पर, केवल 2^(32-28) - 2 = 14
आईपी पते उपलब्ध हैं। `kubernetes.default` सर्विस हमेशा बनाई जाती है; इस उदाहरण के लिए,
यह आपको केवल 13 संभावित सेवाओं के साथ छोड़ देता है।

```sh
for i in $(seq 1 13); do kubectl create service clusterip "test-$i" --tcp 80 -o json | jq -r .spec.clusterIP; done
```

```
10.96.0.11
10.96.0.5
10.96.0.12
10.96.0.13
10.96.0.14
10.96.0.2
10.96.0.3
10.96.0.4
10.96.0.6
10.96.0.7
10.96.0.8
10.96.0.9
error: failed to create ClusterIP service: Internal error occurred: failed to allocate a serviceIP: range is full
```

आप सेवाओं के लिए उपलब्ध आईपी पतों की संख्या बढ़ा सकते हैं, एक नया ServiceCIDR बनाकर
जो आईपी पता रेंज का विस्तार करता है या जोड़ता है।

```sh
cat <EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: ServiceCIDR
metadata:
  name: newcidr1
spec:
  cidrs:
  - 10.96.0.0/24
EOF
```

```
servicecidr.networking.k8s.io/newcidr1 created
```

और यह आपको इस नई रेंज से चुने गए ClusterIPs के साथ नई सेवाएं बनाने की अनुमति देगा।

```sh
for i in $(seq 13 16); do kubectl create service clusterip "test-$i" --tcp 80 -o json | jq -r .spec.clusterIP; done
```

```
10.96.0.48
10.96.0.200
10.96.0.121
10.96.0.144
```

### एक ServiceCIDR को हटाना

आप एक ServiceCIDR को नहीं हटा सकते हैं यदि IPAddresses हैं जो ServiceCIDR पर निर्भर करते हैं।

```sh
kubectl delete servicecidr newcidr1
```

```
servicecidr.networking.k8s.io "newcidr1" deleted
```

कुबेरनेट्स इस आश्रित संबंध को ट्रैक करने के लिए ServiceCIDR पर एक फाइनलाइज़र का उपयोग करता है।

```sh
kubectl get servicecidr newcidr1 -o yaml
```

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: ServiceCIDR
metadata:
  creationTimestamp: "2023-10-12T15:11:07Z"
  deletionGracePeriodSeconds: 0
  deletionTimestamp: "2023-10-12T15:12:45Z"
  finalizers:
  - networking.k8s.io/service-cidr-finalizer
  name: newcidr1
  resourceVersion: "1133"
  uid: 5ffd8afe-c78f-4e60-ae76-cec448a8af40
spec:
  cidrs:
  - 10.96.0.0/24
status:
  conditions:
  - lastTransitionTime: "2023-10-12T15:12:45Z"
    message: There are still IPAddresses referencing the ServiceCIDR, please remove
      them or create a new ServiceCIDR
    reason: OrphanIPAddress
    status: "False"
    type: Ready
```

उन सेवाओं को हटाकर जिनमें वे आईपी पते हैं जो ServiceCIDR को हटाने से रोक रहे हैं

```sh
for i in $(seq 13 16); do kubectl delete service "test-$i" ; done
```

```
service "test-13" deleted
service "test-14" deleted
service "test-15" deleted
service "test-16" deleted
```

कंट्रोल प्लेन हटाने को नोटिस करता है। कंट्रोल प्लेन फिर अपने फाइनलाइज़र को हटा देता है,
ताकि जो ServiceCIDR हटाने के लिए लंबित था वह वास्तव में हटा दिया जाएगा।

```sh
kubectl get servicecidr newcidr1
```

```
Error from server (NotFound): servicecidrs.networking.k8s.io "newcidr1" not found
```

## कुबेरनेट्स सर्विस CIDR नीतियां

क्लस्टर प्रशासक क्लस्टर के भीतर ServiceCIDR संसाधनों के निर्माण और
संशोधन को नियंत्रित करने के लिए नीतियां लागू कर सकते हैं। यह सेवाओं के लिए उपयोग किए जाने वाले आईपी पता रेंज के
केंद्रीकृत प्रबंधन की अनुमति देता है और
अनजाने या परस्पर विरोधी कॉन्फ़िगरेशन को रोकने में मदद करता है। कुबेरनेट्स इन नियमों को लागू करने के लिए मान्य प्रवेश नीतियों जैसे तंत्र प्रदान करता है।

### मान्य प्रवेश नीति का उपयोग करके अनधिकृत ServiceCIDR निर्माण/अद्यतन को रोकना

ऐसी स्थितियाँ हो सकती हैं जब क्लस्टर प्रशासक उन
रेंजों को प्रतिबंधित करना चाहते हैं जिन्हें अनुमति दी जा सकती है या क्लस्टर
सर्विस आईपी रेंज में किसी भी बदलाव को पूरी तरह से अस्वीकार करना चाहते हैं।

{{< note >}}
डिफ़ॉल्ट "kubernetes" ServiceCIDR kube-apiserver द्वारा
क्लस्टर में स्थिरता प्रदान करने के लिए बनाया गया है और क्लस्टर के काम करने के लिए आवश्यक है,
इसलिए इसे हमेशा अनुमति दी जानी चाहिए। आप यह सुनिश्चित कर सकते हैं कि आपकी `ValidatingAdmissionPolicy`
डिफ़ॉल्ट ServiceCIDR को क्लॉज जोड़कर प्रतिबंधित नहीं करती है:

```yaml
  matchConditions:
  - name: 'exclude-default-servicecidr'
    expression: "object.metadata.name != 'kubernetes'"
```

जैसा कि नीचे दिए गए उदाहरणों में है।
{{< /note >}}

#### कुछ विशिष्ट श्रेणियों के लिए सर्विस CIDR रेंज को प्रतिबंधित करें

निम्नलिखित `ValidatingAdmissionPolicy` का एक उदाहरण है जो केवल ServiceCIDRs को
बनाने की अनुमति देता है यदि वे दिए गए `allowed` रेंज के सबरेंज हैं।
(तो उदाहरण नीति `cidrs: ['10.96.1.0/24']`
या `cidrs: ['2001:db8:0:0:ffff::/80', '10.96.0.0/20']` के साथ एक ServiceCIDR की अनुमति देगी, लेकिन `cidrs: ['172.20.0.0/16']` के साथ एक
ServiceCIDR की अनुमति नहीं देगी।) आप इस नीति की प्रतिलिपि बना सकते हैं और
`allowed` का मान अपने क्लस्टर के लिए उपयुक्त कुछ में बदल सकते हैं।

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "servicecidrs.default"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups:   ["networking.k8s.io"]
        apiVersions: ["v1beta1"]
        operations:  ["CREATE", "UPDATE"]
        resources:   ["servicecidrs"]
  validations:
    - expression: |
        !has(object.spec.cidrs) || object.spec.cidrs.all(cidr,
          variables.allowed.any(allowedRange, cidr.overlaps(allowedRange)))
      message: "the ServiceCIDR network address space must be a subrange of the allowed ranges"
      variables:
        - name: allowed
          value:
          - "10.96.0.0/12"
          - "2001:db8::/32"
  matchConditions:
  - name: 'exclude-default-servicecidr'
    expression: "object.metadata.name != 'kubernetes'"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "servicecidrs.default"
spec:
  policyName: "servicecidrs.default"
  validationActions: [Deny]
``` 