---
layout:     post
title:      Golang client-go 源码分析
subtitle:   
date:       2020-04-10
author:     Frederick
header-img: img/bk_go.jpg
catalog: true
tags:
    - Golang
---

# client-go 源码分析

## client-go 简介

client-go是一个调用kubernetes集群资源对象API的客户端，即通过client-go实现对kubernetes集群中资源对象（包括deployment、service、ingress、replicaSet、pod、namespace、node等）的增删改查等操作。大部分对kubernetes进行前置API封装的二次开发都通过client-go这个第三方包来实现。

​client-go官方文档：https://github.com/kubernetes/client-go

## client-go 源码目录结构

- The **kubernetes** package contains the clientset to access Kubernetes API.
- The **discovery** package is used to discover APIs supported by a Kubernetes API server.
- The **dynamic** package contains a dynamic client that can perform generic operations on arbitrary Kubernetes API objects.
- The **transport** package is used to set up auth and start a connection.
- The **tools/cache** package is useful for writing controllers.

### 示例代码

    #demo.go
    package main

    import (
        "flag"
        "fmt"
        "os"
        "path/filepath"
        "time"

        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/client-go/kubernetes"
        "k8s.io/client-go/tools/clientcmd"
    )

    func main() {
        kubeconfig := ./config
        config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
        if err != nil {
            panic(err.Error())
        }
        // creates the clientset
        clientset, err := kubernetes.NewForConfig(config)
        if err != nil {
            panic(err.Error())
        }
        pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
        if err != nil {
            panic(err.Error())
        }
        fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
    }

client-go库版本

    go get  k8s.io/client-go v0.0.0-20191114101535-6c5935290e33

#### kubeconfig

获取kubernetes配置文件kubeconfig的绝对路径。一般路径为$HOME/.kube/config。该文件主要用来配置本地连接的kubernetes集群。config配置文件支持集群内和集群外访问方式。(只要在网络策略访问范围内)

**config文件形式如下所示：**


    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: { .Certificate-authority-data}
        server: https://ip:6443
    name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: kubernetes-admin
    name: kubernetes-admin@kubernetes
    current-context: kubernetes-admin@kubernetes
    kind: Config
    preferences: {}
    users:
    - name: kubernetes-admin
    user:
        client-certificate-data: { .Client-certificate-data}
        client-key-data: { .Client-key-data}

#### rest.config 对象 

    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)

通过参数（master的url或者kubeconfig路径）用BuildConfigFromFlags方法来获取rest.Config对象，一般是通过参数kubeconfig的路径。

### Clientset对象创建

    clientset, err := kubernetes.NewForConfig(config)

通过config对象为入参，调用NewForConfig函数获取clients对象，clients是多个client的集合。里面包含着各个版本的client。源码地址：k8s.io/client-go/kubernetes/clientset.go

    // NewForConfig creates a new Clientset for the given config.
    // If config's RateLimiter is not set and QPS and Burst are acceptable,
    // NewForConfig will generate a rate-limiter in configShallowCopy.
    func NewForConfig(c *rest.Config) (*Clientset, error) {
        configShallowCopy := *c
        if configShallowCopy.RateLimiter == nil && configShallowCopy.QPS > 0 {
            if configShallowCopy.Burst <= 0 {
                return nil, fmt.Errorf("Burst is required to be greater than 0 when RateLimiter is not set and QPS is set to greater than 0")
            }
            configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
        }
        var cs Clientset
        var err error
        cs.admissionregistrationV1, err = admissionregistrationv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.admissionregistrationV1beta1, err = admissionregistrationv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.appsV1, err = appsv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.appsV1beta1, err = appsv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.appsV1beta2, err = appsv1beta2.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.auditregistrationV1alpha1, err = auditregistrationv1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.authenticationV1, err = authenticationv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.authenticationV1beta1, err = authenticationv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.authorizationV1, err = authorizationv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.authorizationV1beta1, err = authorizationv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.autoscalingV1, err = autoscalingv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.autoscalingV2beta1, err = autoscalingv2beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.autoscalingV2beta2, err = autoscalingv2beta2.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.batchV1, err = batchv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.batchV1beta1, err = batchv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.batchV2alpha1, err = batchv2alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.certificatesV1beta1, err = certificatesv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.coordinationV1beta1, err = coordinationv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.coordinationV1, err = coordinationv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        //coreV1 对象创建
        cs.coreV1, err = corev1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.discoveryV1alpha1, err = discoveryv1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.eventsV1beta1, err = eventsv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.extensionsV1beta1, err = extensionsv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.networkingV1, err = networkingv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.networkingV1beta1, err = networkingv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.nodeV1alpha1, err = nodev1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.nodeV1beta1, err = nodev1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.policyV1beta1, err = policyv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.rbacV1, err = rbacv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.rbacV1beta1, err = rbacv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.rbacV1alpha1, err = rbacv1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.schedulingV1alpha1, err = schedulingv1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.schedulingV1beta1, err = schedulingv1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.schedulingV1, err = schedulingv1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.settingsV1alpha1, err = settingsv1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.storageV1beta1, err = storagev1beta1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.storageV1, err = storagev1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        cs.storageV1alpha1, err = storagev1alpha1.NewForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }

        cs.DiscoveryClient, err = discovery.NewDiscoveryClientForConfig(&configShallowCopy)
        if err != nil {
            return nil, err
        }
        return &cs, nil
    }

NewForConfig函数根据传入的rest.config对象创建一个Clientset对象,此对象可以操作CoreV1()方法。

#### Clientset 结构体


**其结构体具体参数如下所示：**

    // Clientset contains the clients for groups. Each group has exactly one
    // version included in a Clientset.
    type Clientset struct {
        *discovery.DiscoveryClient
        admissionregistrationV1      *admissionregistrationv1.AdmissionregistrationV1Client
        admissionregistrationV1beta1 *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
        appsV1                       *appsv1.AppsV1Client
        appsV1beta1                  *appsv1beta1.AppsV1beta1Client
        appsV1beta2                  *appsv1beta2.AppsV1beta2Client
        auditregistrationV1alpha1    *auditregistrationv1alpha1.AuditregistrationV1alpha1Client
        authenticationV1             *authenticationv1.AuthenticationV1Client
        authenticationV1beta1        *authenticationv1beta1.AuthenticationV1beta1Client
        authorizationV1              *authorizationv1.AuthorizationV1Client
        authorizationV1beta1         *authorizationv1beta1.AuthorizationV1beta1Client
        autoscalingV1                *autoscalingv1.AutoscalingV1Client
        autoscalingV2beta1           *autoscalingv2beta1.AutoscalingV2beta1Client
        autoscalingV2beta2           *autoscalingv2beta2.AutoscalingV2beta2Client
        batchV1                      *batchv1.BatchV1Client
        batchV1beta1                 *batchv1beta1.BatchV1beta1Client
        batchV2alpha1                *batchv2alpha1.BatchV2alpha1Client
        certificatesV1beta1          *certificatesv1beta1.CertificatesV1beta1Client
        coordinationV1beta1          *coordinationv1beta1.CoordinationV1beta1Client
        coordinationV1               *coordinationv1.CoordinationV1Client
        coreV1                       *corev1.CoreV1Client
        discoveryV1alpha1            *discoveryv1alpha1.DiscoveryV1alpha1Client
        eventsV1beta1                *eventsv1beta1.EventsV1beta1Client
        extensionsV1beta1            *extensionsv1beta1.ExtensionsV1beta1Client
        networkingV1                 *networkingv1.NetworkingV1Client
        networkingV1beta1            *networkingv1beta1.NetworkingV1beta1Client
        nodeV1alpha1                 *nodev1alpha1.NodeV1alpha1Client
        nodeV1beta1                  *nodev1beta1.NodeV1beta1Client
        policyV1beta1                *policyv1beta1.PolicyV1beta1Client
        rbacV1                       *rbacv1.RbacV1Client
        rbacV1beta1                  *rbacv1beta1.RbacV1beta1Client
        rbacV1alpha1                 *rbacv1alpha1.RbacV1alpha1Client
        schedulingV1alpha1           *schedulingv1alpha1.SchedulingV1alpha1Client
        schedulingV1beta1            *schedulingv1beta1.SchedulingV1beta1Client
        schedulingV1                 *schedulingv1.SchedulingV1Client
        settingsV1alpha1             *settingsv1alpha1.SettingsV1alpha1Client
        storageV1beta1               *storagev1beta1.StorageV1beta1Client
        storageV1                    *storagev1.StorageV1Client
        storageV1alpha1              *storagev1alpha1.StorageV1alpha1Client
    }

#### Clientset 实现的接口

    type Interface interface {
        Discovery() discovery.DiscoveryInterface
        AdmissionregistrationV1() admissionregistrationv1.AdmissionregistrationV1Interface
        AdmissionregistrationV1beta1() admissionregistrationv1beta1.AdmissionregistrationV1beta1Interface
        AppsV1() appsv1.AppsV1Interface
        AppsV1beta1() appsv1beta1.AppsV1beta1Interface
        AppsV1beta2() appsv1beta2.AppsV1beta2Interface
        AuditregistrationV1alpha1() auditregistrationv1alpha1.AuditregistrationV1alpha1Interface
        AuthenticationV1() authenticationv1.AuthenticationV1Interface
        AuthenticationV1beta1() authenticationv1beta1.AuthenticationV1beta1Interface
        AuthorizationV1() authorizationv1.AuthorizationV1Interface
        AuthorizationV1beta1() authorizationv1beta1.AuthorizationV1beta1Interface
        AutoscalingV1() autoscalingv1.AutoscalingV1Interface
        AutoscalingV2beta1() autoscalingv2beta1.AutoscalingV2beta1Interface
        AutoscalingV2beta2() autoscalingv2beta2.AutoscalingV2beta2Interface
        BatchV1() batchv1.BatchV1Interface
        BatchV1beta1() batchv1beta1.BatchV1beta1Interface
        BatchV2alpha1() batchv2alpha1.BatchV2alpha1Interface
        CertificatesV1beta1() certificatesv1beta1.CertificatesV1beta1Interface
        CoordinationV1beta1() coordinationv1beta1.CoordinationV1beta1Interface
        CoordinationV1() coordinationv1.CoordinationV1Interface
        CoreV1() corev1.CoreV1Interface
        DiscoveryV1alpha1() discoveryv1alpha1.DiscoveryV1alpha1Interface
        EventsV1beta1() eventsv1beta1.EventsV1beta1Interface
        ExtensionsV1beta1() extensionsv1beta1.ExtensionsV1beta1Interface
        NetworkingV1() networkingv1.NetworkingV1Interface
        NetworkingV1beta1() networkingv1beta1.NetworkingV1beta1Interface
        NodeV1alpha1() nodev1alpha1.NodeV1alpha1Interface
        NodeV1beta1() nodev1beta1.NodeV1beta1Interface
        PolicyV1beta1() policyv1beta1.PolicyV1beta1Interface
        RbacV1() rbacv1.RbacV1Interface
        RbacV1beta1() rbacv1beta1.RbacV1beta1Interface
        RbacV1alpha1() rbacv1alpha1.RbacV1alpha1Interface
        SchedulingV1alpha1() schedulingv1alpha1.SchedulingV1alpha1Interface
        SchedulingV1beta1() schedulingv1beta1.SchedulingV1beta1Interface
        SchedulingV1() schedulingv1.SchedulingV1Interface
        SettingsV1alpha1() settingsv1alpha1.SettingsV1alpha1Interface
        StorageV1beta1() storagev1beta1.StorageV1beta1Interface
        StorageV1() storagev1.StorageV1Interface
        StorageV1alpha1() storagev1alpha1.StorageV1alpha1Interface
    }

Clientset结构体实现了以上结构定义的所有方法。源码地址：k8s.io/client-go/kubernetes/clientset.go

因为Clientset可使用其中任意函数调用，如获取Pod列表。

    pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})

### CoreV1Client对象创建

在创建Clientset对象时，Clientset 中的变量coreV1也被一起初始化创建。即创建了CoreV1Client对象。

        //coreV1 对象创建
        cs.coreV1, err = corev1.NewForConfig(&configShallowCopy)

#### NewForConfig

    // NewForConfig creates a new CoreV1Client for the given config.
    func NewForConfig(c *rest.Config) (*CoreV1Client, error) {
        config := *c
        if err := setConfigDefaults(&config); err != nil {
            return nil, err
        }
        client, err := rest.RESTClientFor(&config)
        if err != nil {
            return nil, err
        }
        return &CoreV1Client{client}, nil
    }


#### CoreV1Interface

如下所示：

    type CoreV1Interface interface {
        RESTClient() rest.Interface
        ComponentStatusesGetter
        ConfigMapsGetter
        EndpointsGetter
        EventsGetter
        LimitRangesGetter
        NamespacesGetter
        NodesGetter
        PersistentVolumesGetter
        PersistentVolumeClaimsGetter
        PodsGetter
        PodTemplatesGetter
        ReplicationControllersGetter
        ResourceQuotasGetter
        SecretsGetter
        ServicesGetter
        ServiceAccountsGetter
    }

CoreV1Interface中包含了各种kubernetes对象的调用接口，例如PodsGetter是对kubernetes中pod对象增删改查操作的接口。ServicesGetter是对service对象的操作的接口。

#### CoreV1Client 结构体

        type CoreV1Client struct {
            restClient rest.Interface
        }

CoreV1Client结构体实现了CoreV1Interface所有的定义函数。

    //CoreV1Client的方法
    func (c *CoreV1Client) ComponentStatuses() ComponentStatusInterface
    //ConfigMaps
    func (c *CoreV1Client) ConfigMaps(namespace string) ConfigMapInterface
    //Endpoints
    func (c *CoreV1Client) Endpoints(namespace string) EndpointsInterface
    func (c *CoreV1Client) Events(namespace string) EventInterface
    func (c *CoreV1Client) LimitRanges(namespace string) LimitRangeInterface
    //namespace
    func (c *CoreV1Client) Namespaces() NamespaceInterface 
    //nodes
    func (c *CoreV1Client) Nodes() NodeInterface 
    //pv
    func (c *CoreV1Client) PersistentVolumes() PersistentVolumeInterface 
    //pvc
    func (c *CoreV1Client) PersistentVolumeClaims(namespace string) PersistentVolumeClaimInterface 
    //pods 
    func (c *CoreV1Client) Pods(namespace string) PodInterface 
    func (c *CoreV1Client) PodTemplates(namespace string) PodTemplateInterface 
    //rc
    func (c *CoreV1Client) ReplicationControllers(namespace string) ReplicationControllerInterface 
    func (c *CoreV1Client) ResourceQuotas(namespace string) ResourceQuotaInterface 
    //secret
    func (c *CoreV1Client) Secrets(namespace string) SecretInterface 
    //service
    func (c *CoreV1Client) Services(namespace string) ServiceInterface 
    func (c *CoreV1Client) ServiceAccounts(namespace string) ServiceAccountInterface 

### PodsGetter接口

    type PodsGetter interface {
        Pods(namespace string) PodInterface
    }

    func (c *CoreV1Client) Pods(namespace string) PodInterface {
        return newPods(c, namespace)
    }

PodsGetter接口中定义了Pods方法此方法返回PodInterface，这样就可以用Clients.Corev1().Pods()方法对Pod进行增删改查操作了。

### Pod对象创建

    func (c *CoreV1Client) Pods(namespace string) PodInterface {
        return newPods(c, namespace)
    }

    // newPods returns a Pods
    func newPods(c *CoreV1Client, namespace string) *pods {
        return &pods{
            client: c.RESTClient(),
            ns:     namespace,
        }
    }

调用Pods方法，再通过newPods函数创建一个Pods的对象。pods对象继承了rest.Interface接口，即最终的实现本质是RESTClient的HTTP调用。

#### PodInterface接口

    // PodInterface has methods to work with Pod resources.
    type PodInterface interface {
        Create(*v1.Pod) (*v1.Pod, error)
        Update(*v1.Pod) (*v1.Pod, error)
        UpdateStatus(*v1.Pod) (*v1.Pod, error)
        Delete(name string, options *metav1.DeleteOptions) error
        DeleteCollection(options *metav1.DeleteOptions, listOptions metav1.ListOptions) error
        Get(name string, options metav1.GetOptions) (*v1.Pod, error)
        List(opts metav1.ListOptions) (*v1.PodList, error)
        Watch(opts metav1.ListOptions) (watch.Interface, error)
        Patch(name string, pt types.PatchType, data []byte, subresources ...string) (result *v1.Pod, err error)
        GetEphemeralContainers(podName string, options metav1.GetOptions) (*v1.EphemeralContainers, error)
        UpdateEphemeralContainers(podName string, ephemeralContainers *v1.EphemeralContainers) (*v1.EphemeralContainers, error)

        PodExpansion
    }

PodInterface接口定义了Pod对象操作的所有方法。

#### Pod结构体

    // pods implements PodInterface
    type pods struct {
        client rest.Interface
        ns     string
    }

Pod对象中继承了rest.Interface，上面提到过此client便是进行http请求调用。

#### Pod List方法

    // List takes label and field selectors, and returns the list of Pods that match those selectors.
    func (c *pods) List(opts metav1.ListOptions) (result *v1.PodList, err error) {
        var timeout time.Duration
        if opts.TimeoutSeconds != nil {
            timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
        }
        result = &v1.PodList{}
        err = c.client.Get().
            Namespace(c.ns).
            Resource("pods").
            VersionedParams(&opts, scheme.ParameterCodec).
            Timeout(timeout).
            Do().
            Into(result)
        return
    }

以上分析了clientset.CoreV1().Pods("").List(metav1.ListOptions{})对pod资源获取的过程，最终是调用RESTClient的方法实现。

### rest client 对象创建

    // NewForConfig creates a new CoreV1Client for the given config.
    func NewForConfig(c *rest.Config) (*CoreV1Client, error) {
        config := *c
        if err := setConfigDefaults(&config); err != nil {
            return nil, err
        }
        client, err := rest.RESTClientFor(&config)
        if err != nil {
            return nil, err
        }
        return &CoreV1Client{client}, nil
    }
 
在CoreV1Client对象创建的时候也根据config对象调用est.RESTClientFor(&config)函数创建了rest client对象。在创建Pod时将CoreV1Client对象的restClient赋值给Pod的client。

#### rest.Interface

    type Interface interface {
        GetRateLimiter() flowcontrol.RateLimiter
        Verb(verb string) *Request
        Post() *Request
        Put() *Request
        Patch(pt types.PatchType) *Request
        Get() *Request
        Delete() *Request
        APIVersion() schema.GroupVersion
    }

此接口定义了http请求的方法。

#### RESTClient结构体

    type RESTClient struct {
        // base is the root URL for all invocations of the client
        base *url.URL
        // versionedAPIPath is a path segment connecting the base URL to the resource root
        versionedAPIPath string

        // contentConfig is the information used to communicate with the server.
        contentConfig ContentConfig

        // serializers contain all serializers for underlying content type.
        serializers Serializers

        // creates BackoffManager that is passed to requests.
        createBackoffMgr func() BackoffManager

        // TODO extract this into a wrapper interface via the RESTClient interface in kubectl.
        Throttle flowcontrol.RateLimiter

        // Set specific behavior of the client.  If not set http.DefaultClient will be used.
        Client *http.Client
    }


#### RESTClient对象实现的接口函数

    func (c *RESTClient) Verb(verb string) *Request {
        backoff := c.createBackoffMgr()

        if c.Client == nil {
            return NewRequest(nil, verb, c.base, c.versionedAPIPath, c.contentConfig, c.serializers, backoff, c.Throttle, 0)
        }
        return NewRequest(c.Client, verb, c.base, c.versionedAPIPath, c.contentConfig, c.serializers, backoff, c.Throttle, c.Client.Timeout)
    }

    // Post begins a POST request. Short for c.Verb("POST").
    func (c *RESTClient) Post() *Request {
        return c.Verb("POST")
    }

    // Put begins a PUT request. Short for c.Verb("PUT").
    func (c *RESTClient) Put() *Request {
        return c.Verb("PUT")
    }

    // Patch begins a PATCH request. Short for c.Verb("Patch").
    func (c *RESTClient) Patch(pt types.PatchType) *Request {
        return c.Verb("PATCH").SetHeader("Content-Type", string(pt))
    }

    // Get begins a GET request. Short for c.Verb("GET").
    func (c *RESTClient) Get() *Request {
        return c.Verb("GET")
    }

    // Delete begins a DELETE request. Short for c.Verb("DELETE").
    func (c *RESTClient) Delete() *Request {
        return c.Verb("DELETE")
    }

    // APIVersion returns the APIVersion this RESTClient is expected to use.
    func (c *RESTClient) APIVersion() schema.GroupVersion {
        return *c.contentConfig.GroupVersion
    }


通过以上实现可以看出对着的接口调用都转到了Verb方法的调用。Verb方法通过传参调用NewRequest函数最终执行了一次http请求操作。

#### NewRequest

    // NewRequest creates a new request helper object for accessing runtime.Objects on a server.
    func NewRequest(client HTTPClient, verb string, baseURL *url.URL, versionedAPIPath string, content ContentConfig, serializers Serializers, backoff BackoffManager, throttle flowcontrol.RateLimiter, timeout time.Duration) *Request {
        if backoff == nil {
            klog.V(2).Infof("Not implementing request backoff strategy.")
            backoff = &NoBackoff{}
        }

        pathPrefix := "/"
        if baseURL != nil {
            pathPrefix = path.Join(pathPrefix, baseURL.Path)
        }
        r := &Request{
            client:      client,
            verb:        verb,
            baseURL:     baseURL,
            pathPrefix:  path.Join(pathPrefix, versionedAPIPath),
            content:     content,
            serializers: serializers,
            backoffMgr:  backoff,
            throttle:    throttle,
            timeout:     timeout,
        }
        switch {
        case len(content.AcceptContentTypes) > 0:
            r.SetHeader("Accept", content.AcceptContentTypes)
        case len(content.ContentType) > 0:
            r.SetHeader("Accept", content.ContentType+", */*")
        }
        return r
    }

可以看到NewRequest最终将参数组成http请求参数进行了http请求调用。至此clientset.CoreV1().Pods("").List(metav1.ListOptions{})调用完成，最终将结果返回。

## 总结

client-go对kubernetes资源对象的调用操作，需要先获取kubernetes的配置信息，即$HOME/.kube/config。(master节点)

具体流程如下图所示：

![client-go-request.png](https://upload-images.jianshu.io/upload_images/17904159-18812fcf012e02b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## client-go对k8s资源的调用

### Service

    Clientset.CoreV1().Services(nameSpace).Create(serviceObj)
    Clientset.CoreV1().Services(nameSpace).Update(serviceObj)
    Clientset.CoreV1().Services(nameSpace).Delete(serviceName, &meta_v1.DeleteOptions{})
    Clientset.CoreV1().Services(nameSpace).Get(serviceName, meta_v1.GetOptions{})
    Clientset.CoreV1().Services(nameSpace).List(meta_v1.ListOptions{})
    Clientset.CoreV1().Services(nameSpace).Watch(meta_v1.ListOptions{})

### Deployment

    Clientset.AppsV1beta1().Deployments(nameSpace).Create(deploymentObj)
    Clientset.AppsV1beta1().Deployments(nameSpace).Update(deploymentObj)
    Clientset.AppsV1beta1().Deployments(nameSpace).Delete(deploymentName, &meta_v1.DeleteOptions{})
    Clientset.AppsV1beta1().Deployments(nameSpace).Get(deploymentName, meta_v1.GetOptions{})
    Clientset.AppsV1beta1().Deployments(nameSpace).List(meta_v1.ListOptions{})
    Clientset.AppsV1beta1().Deployments(nameSpace).Watch(meta_v1.ListOptions{})

### ReplicaSet

    Clientset.ExtensionsV1beta1().ReplicaSets(nameSpace).Create(replicasetsObj)
    Clientset.ExtensionsV1beta1().ReplicaSets(nameSpace).Update(replicasetsObj)
    Clientset.ExtensionsV1beta1().ReplicaSets(nameSpace).Delete(replicaSetName, &meta_v1.DeleteOptions{})
    Clientset.ExtensionsV1beta1().ReplicaSets(nameSpace).Get(replicaSetName, meta_v1.GetOptions{})
    Clientset.ExtensionsV1beta1().ReplicaSets(nameSpace).List(meta_v1.ListOptions{})
    Clientset.ExtensionsV1beta1().ReplicaSets(nameSpace).Watch(meta_v1.ListOptions{})

### Ingresse

    Clientset.ExtensionsV1beta1().Ingresses(nameSpace).Create(ingressObj)
    Clientset.ExtensionsV1beta1().Ingresses(nameSpace).Update(ingressObj)
    Clientset.ExtensionsV1beta1().Ingresses(nameSpace).Delete(ingressName, &meta_v1.DeleteOptions{})
    Clientset.ExtensionsV1beta1().Ingresses(nameSpace).Get(ingressName, meta_v1.GetOptions{})
    Clientset.ExtensionsV1beta1().Ingresses(nameSpace).List(meta_v1.ListOptions{})

### StatefulSet

    Clientset.AppsV1beta1().StatefulSets(nameSpace).Create(statefulSetObj)
    Clientset.AppsV1beta1().StatefulSets(nameSpace).Update(statefulSetObj)
    Clientset.AppsV1beta1().StatefulSets(nameSpace).Delete(statefulSetName, &meta_v1.DeleteOptions{})
    Clientset.AppsV1beta1().StatefulSets(nameSpace).Get(statefulSetName, meta_v1.GetOptions{})
    Clientset.AppsV1beta1().StatefulSets(nameSpace).List(meta_v1.ListOptions{})

### DaemonSet

    Clientset.ExtensionsV1beta1().DaemonSets(nameSpace).Create(daemonSetObj)
    Clientset.ExtensionsV1beta1().DaemonSets(nameSpace).Update(daemonSetObj)
    Clientset.ExtensionsV1beta1().DaemonSets(nameSpace).Delete(daemonSetName, &meta_v1.DeleteOptions{})
    Clientset.ExtensionsV1beta1().DaemonSets(nameSpace).Get(daemonSetName, meta_v1.GetOptions{})
    Clientset.ExtensionsV1beta1().DaemonSets(nameSpace).List(meta_v1.ListOptions{})

### ReplicationController

    Clientset.CoreV1().ReplicationControllers(nameSpace).Create(replicationControllerObj)
    Clientset.CoreV1().ReplicationControllers(nameSpace).Update(replicationControllerObj)
    Clientset.CoreV1().ReplicationControllers(nameSpace).Delete(replicationControllerName, &meta_v1.DeleteOptions{})
    Clientset.CoreV1().ReplicationControllers(nameSpace).Get(replicationControllerName, meta_v1.GetOptions{})
    Clientset.CoreV1().ReplicationControllers(nameSpace).List(meta_v1.ListOptions{})

### Secret

    Clientset.CoreV1().Secrets(nameSpace).Create(secretObj)
    Clientset.CoreV1().Secrets(nameSpace).Update(secretObj)
    Clientset.CoreV1().Secrets(nameSpace).Delete(secretName, &meta_v1.DeleteOptions{})
    Clientset.CoreV1().Secrets(nameSpace).Get(secretName, meta_v1.GetOptions{})
    Clientset.CoreV1().Secrets(nameSpace).List(meta_v1.ListOptions{})

### ConfigMap

    Clientset.CoreV1().ConfigMaps(nameSpace).Create(configMapObj)
    Clientset.CoreV1().ConfigMaps(nameSpace).Update(configMapObj)
    Clientset.CoreV1().ConfigMaps(nameSpace).Delete(configMapName, &meta_v1.DeleteOptions{})
    Clientset.CoreV1().ConfigMaps(nameSpace).Get(configMapName, meta_v1.GetOptions{})
    Clientset.CoreV1().ConfigMaps(nameSpace).List(meta_v1.ListOptions{})

### Pod

    Clientset.CoreV1().Pods(nameSpace).Get(podName, meta_v1.GetOptions{})
    Clientset.CoreV1().Pods(nameSpace).List(meta_v1.ListOptions{})
    Clientset.CoreV1().Pods(nameSpace).Delete(podName, &meta_v1.DeleteOptions{})

### Namespace

    Clientset.CoreV1().Namespaces().Create(nsSpec)
    Clientset.CoreV1().Namespaces().Get(nameSpace, meta_v1.GetOptions{})
    Clientset.CoreV1().Namespaces().List(meta_v1.ListOptions{})
