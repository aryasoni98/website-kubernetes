---
reviewers:
- rickypai
- thockin
title: HostAliases के साथ पॉड /etc/hosts में प्रविष्टियाँ जोड़ना
content_type: task
weight: 60
min-kubernetes-server-version: 1.7
---


<!-- overview -->

एक पॉड की `/etc/hosts` फ़ाइल में प्रविष्टियाँ जोड़ने से पॉड-स्तरीय होस्टनाम रिज़ॉल्यूशन का ओवरराइड होता है जब DNS और अन्य विकल्प लागू नहीं होते हैं। आप इन कस्टम प्रविष्टियों को पॉडस्पेक में HostAliases फ़ील्ड के साथ जोड़ सकते हैं।

कुबेरनेट्स प्रोजेक्ट `hostAliases` फ़ील्ड (पॉड के लिए `.spec` का हिस्सा) का उपयोग करके DNS कॉन्फ़िगरेशन को संशोधित करने की सिफारिश करता है, और `/etc/hosts` को सीधे संपादित करने के लिए एक इनिट कंटेनर या अन्य साधनों का उपयोग करके नहीं।
अन्य तरीकों से किए गए परिवर्तन पॉड निर्माण या पुनरारंभ के दौरान कुबेलेट द्वारा अधिलेखित किए जा सकते हैं।

<!-- steps -->

## डिफ़ॉल्ट होस्ट फ़ाइल सामग्री

एक Nginx पॉड शुरू करें जिसे एक पॉड आईपी सौंपा गया है:

```shell
kubectl run nginx --image nginx
```

```
pod/nginx created
```

पॉड आईपी की जांच करें:

```shell
kubectl get pods --output=wide
```

```
NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
nginx    1/1       Running   0          13s    10.200.0.4   worker0
```

होस्ट फ़ाइल सामग्री इस तरह दिखेगी:

```shell
kubectl exec nginx -- cat /etc/hosts
```

```
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.4	nginx
```

डिफ़ॉल्ट रूप से, `hosts` फ़ाइल में केवल IPv4 और IPv6 बॉयलरप्लेट जैसे
`localhost` और उसका अपना होस्टनाम शामिल होता है।

## hostAliases के साथ अतिरिक्त प्रविष्टियाँ जोड़ना

डिफ़ॉल्ट बॉयलरप्लेट के अलावा, आप `hosts` फ़ाइल में अतिरिक्त प्रविष्टियाँ जोड़ सकते हैं।
उदाहरण के लिए: `foo.local`, `bar.local` को `127.0.0.1` और `foo.remote`,
`bar.remote` को `10.1.2.3` पर हल करने के लिए, आप `.spec.hostAliases` के तहत एक पॉड के लिए HostAliases कॉन्फ़िगर कर सकते हैं:

{{% code_sample file="service/networking/hostaliases-pod.yaml" %}}

आप उस कॉन्फ़िगरेशन के साथ एक पॉड शुरू कर सकते हैं:

```shell
kubectl apply -f https://k8s.io/examples/service/networking/hostaliases-pod.yaml
```

```
pod/hostaliases-pod created
```

पॉड के विवरण की जांच करके उसका IPv4 पता और उसकी स्थिति देखें:

```shell
kubectl get pod --output=wide
```

```
NAME                           READY     STATUS      RESTARTS   AGE       IP              NODE
hostaliases-pod                0/1       Completed   0          6s        10.200.0.5      worker0
```

`hosts` फ़ाइल सामग्री इस तरह दिखती है:

```shell
kubectl logs hostaliases-pod
```

```
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.5	hostaliases-pod

# Entries added by HostAliases.
127.0.0.1	foo.local	bar.local
10.1.2.3	foo.remote	bar.remote
```

नीचे निर्दिष्ट अतिरिक्त प्रविष्टियों के साथ।

## कुबेलेट होस्ट फ़ाइल का प्रबंधन क्यों करता है? {#why-does-kubelet-manage-the-hosts-file}

कुबेलेट पॉड के प्रत्येक कंटेनर के लिए `hosts` फ़ाइल का प्रबंधन करता है ताकि कंटेनर रनटाइम को कंटेनर शुरू होने के बाद फ़ाइल को संशोधित करने से रोका जा सके।
ऐतिहासिक रूप से, कुबेरनेट्स ने हमेशा डॉकर इंजन को अपने कंटेनर रनटाइम के रूप में उपयोग किया, और डॉकर इंजन प्रत्येक कंटेनर के शुरू होने के बाद `/etc/hosts` फ़ाइल को संशोधित करता था।

वर्तमान कुबेरनेट्स विभिन्न प्रकार के कंटेनर रनटाइम का उपयोग कर सकता है; फिर भी, कुबेलेट प्रत्येक कंटेनर के भीतर होस्ट फ़ाइल का प्रबंधन करता है ताकि परिणाम इच्छित हो, भले ही आप किस कंटेनर रनटाइम का उपयोग करें।

{{< caution >}}
एक कंटेनर के अंदर होस्ट फ़ाइल में मैन्युअल परिवर्तन करने से बचें।

यदि आप होस्ट फ़ाइल में मैन्युअल परिवर्तन करते हैं,
तो कंटेनर से बाहर निकलने पर वे परिवर्तन खो जाते हैं।
{{< /caution >}} 