---
title: Kustomize का उपयोग करके सीक्रेट्स का प्रबंधन
content_type: task
weight: 30
description: kustomization.yaml फाइल का उपयोग करके सीक्रेट ऑब्जेक्ट बनाना।
---

<!-- overview -->

`kubectl` [Kustomize ऑब्जेक्ट प्रबंधन टूल](/docs/tasks/manage-kubernetes-objects/kustomization/) का उपयोग करके सीक्रेट्स और कॉन्फ़िगमैप्स को प्रबंधित करने का समर्थन करता है। आप Kustomize का उपयोग करके एक *रिसोर्स जनरेटर* बनाते हैं, जो एक सीक्रेट जनरेट करता है जिसे आप `kubectl` का उपयोग करके API सर्वर पर लागू कर सकते हैं।

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}}

<!-- steps -->

## सीक्रेट बनाएं

आप एक `secretGenerator` को `kustomization.yaml` फाइल में परिभाषित करके एक सीक्रेट जनरेट कर सकते हैं जो अन्य मौजूदा फाइलों, `.env` फाइलों, या लिटरल मानों का संदर्भ देता है। उदाहरण के लिए, निम्नलिखित निर्देश उपयोगकर्ता नाम `admin` और पासवर्ड `1f2d1e2e67df` के लिए एक kustomization फाइल बनाते हैं।

{{< note >}}
सीक्रेट के लिए `stringData` फील्ड सर्वर-साइड एप्लाई के साथ अच्छी तरह से काम नहीं करता है।
{{< /note >}}

### kustomization फाइल बनाएं

{{< tabs name="Secret data" >}}
{{< tab name="Literals" codelang="yaml" >}}
secretGenerator:
- name: database-creds
  literals:
  - username=admin
  - password=1f2d1e2e67df
{{< /tab >}}
{{% tab name="Files" %}}
1.  क्रेडेंशियल्स को फाइलों में स्टोर करें। फाइलनाम सीक्रेट की कुंजियां हैं:

    ```shell
    echo -n 'admin' > ./username.txt
    echo -n '1f2d1e2e67df' > ./password.txt
    ```
    `-n` फ्लैग यह सुनिश्चित करता है कि आपकी फाइलों के अंत में कोई न्यूलाइन वर्ण नहीं है।

1.  `kustomization.yaml` फाइल बनाएं:

    ```yaml
    secretGenerator:
    - name: database-creds
      files:
      - username.txt
      - password.txt
    ```
{{% /tab %}}
{{% tab name=".env files" %}}
आप `.env` फाइलें प्रदान करके भी `kustomization.yaml` फाइल में secretGenerator को परिभाषित कर सकते हैं। उदाहरण के लिए, निम्नलिखित `kustomization.yaml` फाइल एक `.env.secret` फाइल से डेटा प्राप्त करती है:

```yaml
secretGenerator:
- name: db-user-pass
  envs:
  - .env.secret
```
{{% /tab %}}
{{< /tabs >}}

सभी मामलों में, आपको मानों को base64 में एनकोड करने की आवश्यकता नहीं है। YAML फाइल का नाम **अवश्य** `kustomization.yaml` या `kustomization.yml` होना चाहिए।

### kustomization फाइल लागू करें

सीक्रेट बनाने के लिए, kustomization फाइल वाली डायरेक्टरी को लागू करें:

```shell
kubectl apply -k <directory-path>
```

आउटपुट इस तरह का होता है:

```
secret/database-creds-5hdh7hhgfk created
```

जब एक सीक्रेट जनरेट किया जाता है, तो सीक्रेट नाम सीक्रेट डेटा को हैश करके और हैश मान को नाम में जोड़कर बनाया जाता है। यह सुनिश्चित करता है कि डेटा को संशोधित करने पर हर बार एक नया सीक्रेट जनरेट किया जाता है।

यह सत्यापित करने के लिए कि सीक्रेट बनाया गया था और सीक्रेट डेटा को डिकोड करने के लिए,

```shell
kubectl get -k <directory-path> -o jsonpath='{.data}' 
```

आउटपुट इस तरह का होता है:

```
{ "password": "MWYyZDFlMmU2N2Rm", "username": "YWRtaW4=" }
```

```
echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
```

आउटपुट इस तरह का होता है:

```
1f2d1e2e67df
```

अधिक जानकारी के लिए, [kubectl का उपयोग करके सीक्रेट्स का प्रबंधन](/docs/tasks/configmap-secret/managing-secret-using-kubectl/#verify-the-secret) और [Kustomize का उपयोग करके कुबेरनेट्स ऑब्जेक्ट्स का घोषणात्मक प्रबंधन](/docs/tasks/manage-kubernetes-objects/kustomization/) देखें।

## सीक्रेट को संपादित करें {#edit-secret}

1.  अपनी `kustomization.yaml` फाइल में, डेटा को संशोधित करें, जैसे `password`।
1.  kustomization फाइल वाली डायरेक्टरी को लागू करें:

    ```shell
    kubectl apply -k <directory-path>
    ```

    आउटपुट इस तरह का होता है:

    ```
    secret/db-user-pass-6f24b56cc8 created
    ```

संपादित सीक्रेट मौजूदा `Secret` ऑब्जेक्ट को अपडेट करने के बजाय एक नए `Secret` ऑब्जेक्ट के रूप में बनाया जाता है। आपको अपने पॉड्स में सीक्रेट के संदर्भों को अपडेट करने की आवश्यकता हो सकती है।

## साफ़ करें

सीक्रेट को हटाने के लिए, `kubectl` का उपयोग करें:

```shell
kubectl delete secret db-user-pass
```

## {{% heading "whatsnext" %}}

- [सीक्रेट कॉन्सेप्ट](/docs/concepts/configuration/secret/) के बारे में और पढ़ें
- [kubectl का उपयोग करके सीक्रेट्स का प्रबंधन](/docs/tasks/configmap-secret/managing-secret-using-kubectl/) करना सीखें
- [कॉन्फ़िग फाइल का उपयोग करके सीक्रेट्स का प्रबंधन](/docs/tasks/configmap-secret/managing-secret-using-config-file/) करना सीखें 