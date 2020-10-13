# Progress <code>[2/5]</code>

-   [X] APISnoop org-flow : [MyEndpoint.org](https://github.com/cncf/apisnoop/blob/master/tickets/k8s/)
-   [X] test approval issue : [kubernetes/kubernetes#](https://github.com/kubernetes/kubernetes/issues/)
-   [ ] test pr : kuberenetes/kubernetes#
-   [ ] two weeks soak start date : testgrid-link
-   [ ] two weeks soak end date :
-   [ ] test promotion pr : kubernetes/kubernetes#?

# Identifying an untested feature Using APISnoop

According to this APIsnoop query, there are still some remaining ProxyWithPath endpoints which are untested.

with this query you can filter untested endpoints by their category and eligiblity for conformance. e.g below shows a query to find all conformance eligible untested,stable,core endpoints

```sql-mode
SELECT
  endpoint,
  -- k8s_action,
  path,
  -- description,
  kind
  FROM testing.untested_stable_endpoint
  where eligible is true
  and category = 'core'
  and endpoint ilike '%Node%WithPath%'
  order by kind, endpoint desc
  limit 25;
```

```example
               endpoint                |               path                |       kind
---------------------------------------|-----------------------------------|------------------
 connectCoreV1PutNodeProxyWithPath     | /api/v1/nodes/{name}/proxy/{path} | NodeProxyOptions
 connectCoreV1PostNodeProxyWithPath    | /api/v1/nodes/{name}/proxy/{path} | NodeProxyOptions
 connectCoreV1PatchNodeProxyWithPath   | /api/v1/nodes/{name}/proxy/{path} | NodeProxyOptions
 connectCoreV1OptionsNodeProxyWithPath | /api/v1/nodes/{name}/proxy/{path} | NodeProxyOptions
 connectCoreV1HeadNodeProxyWithPath    | /api/v1/nodes/{name}/proxy/{path} | NodeProxyOptions
 connectCoreV1DeleteNodeProxyWithPath  | /api/v1/nodes/{name}/proxy/{path} | NodeProxyOptions
(6 rows)

```

# API Reference and feature documentation

-   [Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/kubernetes-api/)
-   [client-go](https://github.com/kubernetes/client-go/blob/master/kubernetes/typed/core/v1)
-   [Kubernetes 1.19: Node v1 Core: Proxy Operations](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#-strong-proxy-operations-node-v1-core-strong-)

# The mock test

## Test outline

1.  Locate connction details from kubeconfig

2.  Create a http.Client to access the node proxy

3.  Iterate through a list of http methods and check the response from the node

4.  Confirm that each response code is 200 OK

## Test the functionality in Go

```go
package main

import (
  // "encoding/json"
  // "context"
  "flag"
  "fmt"
  "net/http"
  "os"

  // v1 "k8s.io/api/core/v1"
  // "k8s.io/client-go/dynamic"
  // "k8s.io/apimachinery/pkg/runtime/schema"
  // metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  // "k8s.io/client-go/kubernetes"
  "k8s.io/client-go/transport"
  // "k8s.io/apimachinery/pkg/types"
  "k8s.io/client-go/tools/clientcmd"
)

// helper function that mirrors framework.ExpectNoError
func ExpectNoError(err error, msg string) {
  if err != nil {
    errMsg := msg + fmt.Sprintf(" %v\n", err)
    os.Stderr.WriteString(errMsg)
    os.Exit(1)
  }
}

// helper function that mirrors framework.ExpectEqual
func ExpectEqual(a int, b int, msg string, i interface{}) {
  if a != b {
    errMsg := msg + fmt.Sprintf(" %v\n", i)
    os.Stderr.WriteString(errMsg)
    os.Exit(1)
  }
}

// helper function to inspect various interfaces
func inspect(level int, name string, i interface{}) {
  fmt.Printf("Inspecting: %s\n", name)
  fmt.Printf("Inspect level: %d   Type: %T\n", level, i)
  switch level {
  case 1:
    fmt.Printf("%+v\n\n", i)
  case 2:
    fmt.Printf("%#v\n\n", i)
  default:
    fmt.Printf("%v\n\n", i)
  }
}

func main() {
  // uses the current context in kubeconfig
  kubeconfig := flag.String("kubeconfig", fmt.Sprintf("%v/%v/%v", os.Getenv("HOME"), ".kube", "config"), "(optional) absolute path to the kubeconfig file")
  flag.Parse()
  config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
  ExpectNoError(err, "Could not build config from flags")
  // make our work easier to find in the audit_event queries
  config.UserAgent = "live-test-writing"
  // creates the clientset
  // ClientSet, _ := kubernetes.NewForConfig(config)
  // DynamicClientSet, _ := dynamic.NewForConfig(config)
  // podResource := schema.GroupVersionResource{Group: "", Version: "v1", Resource: "pods"}

  // TEST BEGINS HERE

  transportCfg, err := config.TransportConfig()
  ExpectNoError(err, "Error creating transportCfg")
  restTransport, err := transport.New(transportCfg)
  ExpectNoError(err, "Error creating restTransport")

  client := &http.Client{
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
      return http.ErrUseLastResponse
    },
    Transport: restTransport,
  }

  httpVerbs := []string{"DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"}
  for _, httpVerb := range httpVerbs {

    urlString := config.Host + "/api/v1/nodes/heyste-humacs-control-plane-vmbww/proxy/configz"
    fmt.Printf("Starting http.Client for %s\n", urlString)
    request, err := http.NewRequest(httpVerb, urlString, nil)
    ExpectNoError(err, "processing request")

    resp, err := client.Do(request)
    ExpectNoError(err, "processing response")
    defer resp.Body.Close()

    fmt.Printf("http.Client request:%s StatusCode:%d\n", httpVerb, resp.StatusCode)
    ExpectEqual(resp.StatusCode, 200, "The resp.StatusCode returned: %d", resp.StatusCode)
  }

  // TEST ENDS HERE

  fmt.Println("[status] complete")

}
```

# Verifying increase in coverage with APISnoop

## Reset Stats

```sql-mode
delete from testing.audit_event;
```

```example
DELETE 62870
```

## Discover useragents:

```sql-mode
select distinct useragent
  from testing.audit_event
 where useragent like 'live%';
```

```example
     useragent
-------------------
 live-test-writing
(1 row)

```

## List endpoints hit by the test:

```sql-mode
select * from testing.endpoint_hit_by_new_test ORDER BY hit_by_ete;
```

```example
     useragent     |               endpoint               | hit_by_ete | hit_by_new_test
-------------------|--------------------------------------|------------|-----------------
 live-test-writing | connectCoreV1DeleteNodeProxyWithPath | f          |               3
 live-test-writing | connectCoreV1PatchNodeProxyWithPath  | f          |               3
 live-test-writing | connectCoreV1PostNodeProxyWithPath   | f          |               3
 live-test-writing | connectCoreV1PutNodeProxyWithPath    | f          |               3
 live-test-writing | connectCoreV1GetNodeProxyWithPath    | t          |               6
(5 rows)

```

## Display endpoint coverage change:

```sql-mode
select * from testing.projected_change_in_coverage;
```

```example
   category    | total_endpoints | old_coverage | new_coverage | change_in_number
---------------|-----------------|--------------|--------------|------------------
 test_coverage |             831 |          306 |          310 |                4
(1 row)

```

# Final notes

If a test with these calls gets merged, **test coverage will go up by 4 points**

This test is also created with the goal of conformance promotion.

---

/sig testing

/sig architecture

/area conformance