# 將 Ingress 資源透過 Envoy Gateway 取代

<!-- markdownlint-disable MD028 -->
<!-- markdownlint-disable MD033 -->

從官方文件中可以知道 Ingress 本身已經不再繼續更新，雖然可以繼續使用，但未來若有重大漏洞，對安全性上會造成很大的威脅。

既然官方現在主推 Gateway API，那將 Ingress 取代為 Gateway 本身就變得極為重要，本專案會說明如何部署 Envoy Gateway 到 Kubernetes 中。

> 使用前可以先閱讀 [Kubernetes Gateway API 文檔](https://kubernetes.io/docs/concepts/services-networking/gateway/)與 [SIG 對 Gateway API 的說明](https://gateway-api.sigs.k8s.io/)

> 請注意，所有的 Helm Charts 檔案請由官方的 OCI 拉取，本儲存庫不提供任何 Charts 檔案，僅提供從官方修改的測試部署 yaml 檔

## Table of Contents

文件與研究未完成，待完成後再補完

## 前置準備

在部署前須先安裝以下工具

- kubectl
- helm

## 安裝

請依據以下說明進行安裝

1. 先透過指令以下指令將 helm chart 拉下來並解壓縮

    > 若沒有修改 `values.yaml` 的需求，其實也可以直接透過以下指令完成安裝，完成後可以直接跳到 4.
    >
    > ```bash
    > helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm -n envoy-gateway-system --create-namespace
    > ```

    ```bash
    helm pull oci://docker.io/envoyproxy/gateway-helm
    ```

2. 修改 `values.yaml`
    - 修改 `global.imageRegistry` 設定，若叢集可對外連線則不用調整
    - 若 `Image` 的版本標籤為 `latest` 則 `pullPolicy` 建議調整為 `Always`，否則可維持 `IfNotPresent` 即可
3. 透過以下指令將 `envoy-gateway` 安裝到 Kubernetes 中

    ```bash
    helm install envoy-gateway ./ -n envoy-gateway-system --create-namespace
    ```

4. 透過指令 `kubectl get pods -n envoy-gateway-system --watch` 待所有的 Pod 狀態都為 Up 表示部署成功

## 建立 GatewayClass

GatewayClass 用於控制哪個 Gateway 來處理流量，一個 GatewayClass 僅可對應到一個 Gateway 的實作，其 yaml 檔如下所示

```yaml
# gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

透過 `kubectl apply -f gateway-class.yaml` 將此 GatewayClass 部署上去後，未來只要 Gateway 指定給此 GatewayClass，流量就會由 Envoy Gateway 處理

## 建立 Gateway

Gateway 用於定義實際監聽的域名、IP 與連接埠，TLS 相關的設定也可以在這邊設定

需特別注意的是如果手邊有多個域名，這邊指定了 Hostname 後，這個 Gateway 就只會針對該域名的流量進行處理，下面是一個具備 TLS 組態的 Gateway

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-gateway
spec:
  gatewayClassName: envoy-gateway-demo-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            group: ""
            name: example-cert
            namespace: envoy-gateway-system
      allowRoutes:
        namespaces:
          from: Same
          # Using below if you want to process traffic to/from other ns
          # from: Selector
          # selector:
          #   matchLabels:
          #     kubernetes.io/metadata.name: some-ns
```

## 建立 HTTPRoute 或 TLSRoute

若在 Gateway 就做 TLS Terminate 的話，基本上後面只要部署 HTTPRoute 就可以了，除非很特殊的情境下在 Gateway 不做 TLS Termination，那後面就必須部署 TLSRoute 來串接流量

這邊的 hostname 也可以做流量過濾，若指定後，將只針對特定域名的流量做處理

- HTTPRoute

  ```yaml
  # http-route.yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: example-backend
    namespace: default
  spec:
    parentRefs:
      - name: envoy-gateway
    # Enable below setting if you want to process specific domain's traffic
    # hostnames:
    #   - "www.example.com"
    rules:
      - backendRefs:
          - group: ""
            kind: Service
            name: backend
            port: 3000
            weight: 1
        matches:
          - path:
              type: PathPrefix
              value: /
  ```

- TLSRoute

  這邊需要注意，Gateway 上對於 TLS 的 mode 需設定為 `passthrough`，否則這個 Route 將不會生效

  ```yaml
  # basic-tlsroute.yaml
  apiVersion: gateway.networking.k8s.io/v1alpha2
  kind: TLSRoute
  metadata:
    name: example-tls-backend
    namespace: default
  spec:
    parentRefs:
      - name: envoy-gateway
    # Enable below setting if you want to process specific domain's traffic
    # hostnames:
    #   - "api.example.com"
    rules:
      - backendRefs:
        - name: api-service
          port: 443
  ```

## 建立 TCPRoute 或 UDPRoute

待測試

## 測試 Envoy Gateway

官方有提供一個[測試的 yaml 檔](https://gateway.envoyproxy.io/docs/tasks/quickstart/#installation)，直接部署後觀察以下幾項資源的狀態是否正常

- Gateway
- HTTPRoute
- TLSRoute

並透過指令 `curl http://YOUR_DOMAIN/YOUR_PATH` 檢查回應，若收到類似於以下的回應表示部署成功

```json
{
  "path": "/YOUR_PATH",
  "host": "YOUR_HOST",
  "method": "GET",
  "proto": "HTTP/1.1",
  "headers": {
    "Accept": ["*/*"],
    "User-Agent": ["curl/7.76.1"],
    "X-Envoy-External-Address": ["10.244.0.1"],
    "X-forwarded-For": ["10.244.0.1"],
    "X-Forwarded-Proto": ["http"],
    "X-Request-Id": ["UUID"]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "eg"
}
```

## ingress-nginx 的 ModSecurity 怎麼辦

Envoy Gateway 可以透過 [WASM 外掛](https://gateway.envoyproxy.io/docs/tasks/extensibility/wasm/)來達到不修改 Envoy Gateway 核心程式碼就可以做到流量過濾的需求

搭配 [Coraza Proxy WASM](https://github.com/corazawaf/coraza-proxy-wasm)就可以很簡單的達到 ModSecurity 的目的

> 請注意，ModSecurity 有些規則還是沒辦法完全對應到 Coraza WAF，且 Coraza 本身也不是隨插即用，需要針對其規則進行調教與篩選

部署方式如下

1. 從 Coraza Proxy Wasm 專案頁面找到 [Package 頁面](https://github.com/corazawaf/coraza-proxy-wasm/pkgs/container/coraza-proxy-wasm)
2. 部署 `EnvoyExtensionPolicy`

    ```yaml
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: EnvoyExtensionPolicy
    metadata:
      name: wasm-test
    spec:
      targetRefs:
      - group: gateway.networking.k8s.io
        kind: HTTPRoute
        name: example-backend
      wasm:
      - name: coraza-waf
        rootID: coraza-filter
        code:
          type: Image
          image:
            url: ghcr.io/corazawaf/coraza-proxy-wasm:0.6.0
        config:
          value: |
            {
              "directives_map": {
                "default": [
                  "SecRuleEngine DetectionOnly",
                  "SecDebugLogLevel 3",
                  "Include @recommended-conf",
                  "Include @crs-setup-conf",
                  "Include @owasp_crs/*.conf"
                ]
              },
              "default_directives": "default"
            }
    ```

3. 透過指令 `curl -i http://YOUR_DOMAIN/YOUR_PATH` 檢查回應中是否飽含有類似於 `x-wasm-custom: FOO` 的 Header 在回應中，若有表示部署成功
4. 持續調教與篩選規則，直到其符合所有安全性規範

## 參考資料

- [Gateway API - Kubernetes 文檔](https://kubernetes.io/zh-cn/docs/concepts/services-networking/gateway/#api-kind-httproute)
- [Quickstart - Envoy Gateway](https://gateway.envoyproxy.io/docs/tasks/quickstart/)
- [Secuure Gateway - Envoy Gateway](https://gateway.envoyproxy.io/docs/tasks/security/secure-gateways/)
- [How to Set Up Kubernetes Gateway API TLSRoute for Passthrough TLS](https://oneuptime.com/blog/post/2026-02-09-gateway-api-tlsroute-passthrough-tls/view)
- [Wasm Extension - Envoy Gateway](https://gateway.envoyproxy.io/docs/tasks/extensibility/wasm/)
- [corazawaf/coraza-proxy-wasm - GitHub](https://github.com/corazawaf/coraza-proxy-wasm)
