---
reviewers:
- sabdullah
- thockin
- dwinship
min-kubernetes-server-version: v1.30
title: डिफ़ॉल्ट सर्विस आईपी रेंज को फिर से कॉन्फ़िगर करें
content_type: task
---

<!-- overview -->

{{< feature-state feature_gate_name="MultiCIDRServiceAllocator" >}}

यह दस्तावेज़ बताता है कि प्रसिद्ध `kubernetes` सर्विस द्वारा उपयोग की जाने वाली डिफ़ॉल्ट आईपी रेंज को कैसे बदला जाए।

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}}

{{< version-check >}}

<!-- steps -->

## डिफ़ॉल्ट सर्विस आईपी रेंज को फिर से कॉन्फ़िगर करें

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

## प्रसिद्ध `kubernetes` सर्विस के आईपी को बदलना

ऐसे मामले हैं जब उपयोगकर्ताओं को `--service-cluster-ip-range` कमांड लाइन तर्क द्वारा परिभाषित डिफ़ॉल्ट सर्विस आईपी रेंज को बदलना पड़ सकता है,
शायद नेटवर्क में एक ओवरलैपिंग आईपी पते के कारण या किसी अन्य कारण से।
अतीत में, सर्विस रेंज को बदलना एक विघटनकारी ऑपरेशन था जो डेटा हानि का कारण भी बन सकता था।
इस नई सुविधा के साथ, उपयोगकर्ता अब `--primary-service-cluster-ip-range` और `--secondary-service-cluster-ip-range` कमांड लाइन तर्कों का उपयोग कर सकते हैं ताकि
आईपी पतों को एक रेंज से दूसरी रेंज में सुरक्षित रूप से स्थानांतरित किया जा सके।

### एक नया ServiceCIDR जोड़ना

सेवाओं के लिए 10.96.0.0/28 रेंज वाले क्लस्टर पर, `kubernetes` सर्विस का आईपी पता `10.96.0.1` है।
यदि आप उस सर्विस के लिए आईपी पता बदलना चाहते हैं, तो आपको एक नया आईपी पता रेंज जोड़ना होगा।
एक उपयोगकर्ता kube-apiserver को `--secondary-service-cluster-ip-range` कमांड लाइन तर्क जोड़कर एक नया आईपी पता रेंज जोड़ सकता है।
यह नया पैरामीटर kube-apiserver को एक और ServiceCIDR ऑब्जेक्ट बनाने के लिए मजबूर करेगा।

यह kube-apiserver को `--secondary-service-cluster-ip-range=10.98.0.0/28` के साथ पुनरारंभ करने के बाद है।
`kubernetes` नाम का ServiceCIDR अभी भी मौजूद है, लेकिन `new-kubernetes-range` नाम का एक नया ServiceCIDR बनाया गया है।

```sh
kubectl get servicecidrs
```

```
NAME                   CIDRS          AGE
kubernetes             10.96.0.0/28   19d
new-kubernetes-range   10.98.0.0/28   1s
```

`kubernetes` सर्विस अभी भी `10.96.0.1` आईपी पते का उपयोग करती है, क्योंकि यह अभी भी मूल ServiceCIDR रेंज में है।

```sh
kubectl get service kubernetes -o jsonpath='{.spec.clusterIP}'
```

```
10.96.0.1
```

अब जब क्लस्टर में दो सर्विस आईपी पता रेंज हैं, तो प्रसिद्ध `kubernetes` सर्विस के आईपी पते को एक रेंज से दूसरी रेंज में स्थानांतरित करना संभव है।

यह kube-apiserver को कमांड लाइन तर्कों के साथ पुनरारंभ करके किया जाता है:
`--service-cluster-ip-range=10.98.0.0/28` (पिछली द्वितीयक रेंज) और `--secondary-service-cluster-ip-range=10.96.0.0/28` (पिछली प्राथमिक रेंज)।

कमांड लाइन तर्कों को स्वैप करने के बाद, `kubernetes` सर्विस का आईपी पता अब `10.98.0.1` है,
जो नई प्राथमिक रेंज से है।

```sh
kubectl get service kubernetes -o jsonpath='{.spec.clusterIP}'
```

```
10.98.0.1
```

ServiceCIDRs के नाम अभी भी समान हैं, लेकिन `kubernetes` ServiceCIDR का CIDR अब `10.98.0.0/28` है,
और `new-kubernetes-range` ServiceCIDR का CIDR अब `10.96.0.0/28` है।

```sh
kubectl get servicecidrs
```

```
NAME                   CIDRS          AGE
kubernetes             10.98.0.0/28   19d
new-kubernetes-range   10.96.0.0/28   2m
```

यह पुष्टि करता है कि माइग्रेशन सफल रहा। अब पुरानी रेंज को हटाना सुरक्षित है।

### पुरानी सर्विस रेंज को हटाना

पुरानी सर्विस रेंज को हटाने के लिए, आपको kube-apiserver से `--secondary-service-cluster-ip-range` कमांड लाइन तर्क को हटाना होगा।
kube-apiserver को पुनरारंभ करने के बाद, `new-kubernetes-range` ServiceCIDR हटा दिया जाएगा।

```sh
kubectl get servicecidrs
```

```
NAME         CIDRS          AGE
kubernetes   10.98.0.0/28   20d
```

यह पुष्टि करता है कि पुरानी रेंज हटा दी गई है और क्लस्टर अब एक नई डिफ़ॉल्ट सर्विस आईपी रेंज के साथ काम कर रहा है।
यह ऑपरेशन विघटनकारी नहीं था और इससे डेटा हानि नहीं हुई।
उपयोगकर्ता अब सेवाओं के लिए आईपी पतों का अनुरोध करना जारी रख सकते हैं और वे नई रेंज से आवंटित किए जाएंगे।
यह नई सुविधा उपयोगकर्ताओं को अपने क्लस्टर आईपी पते की जरूरतों को अधिक कुशलता से और बिना किसी व्यवधान के प्रबंधित करने की अनुमति देती है। 