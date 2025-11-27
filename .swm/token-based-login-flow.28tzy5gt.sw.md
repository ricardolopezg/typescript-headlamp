---
title: Token-Based Login Flow
---
This document describes the flow for logging in with a token. Token-based login allows users to authenticate and gain access to cluster resources by providing a token. The system checks for an available cluster and verifies the user's permissions using the provided token.

```mermaid
flowchart TD
  node1["Token-Based Login Entry Point"]:::HeadingStyle --> node2{"Is cluster available?"}
  click node1 goToHeading "Token-Based Login Entry Point"
  node2 -->|"No"| node4["Login failed"]
  node2 -->|"Yes"| node3["Token Verification via Cluster API"]:::HeadingStyle
  click node3 goToHeading "Token Verification via Cluster API"
  node3 --> node5{"Is authentication successful?"}
  node5 -->|"Yes"| node6["Login successful"]
  node5 -->|"No"| node4
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

```mermaid
graph TD;
      7071df105802ada928e6f2dc24a0786c5eb20f9ea01b03eec196281089daa938(frontend/…/account/Auth.tsx::AuthToken) --> 7c233e049e1faaf21b52f91b1abc7ea25a4700a287ada68bc537787f3d4d7bff(frontend/…/account/Auth.tsx::loginWithToken):::mainFlowStyle

2ed809f7c2bc7c95aea13195a74a179ce3364cdb9d3761f99faabd4d8b99ee92(frontend/…/account/Auth.tsx::onAuthClicked) --> 7c233e049e1faaf21b52f91b1abc7ea25a4700a287ada68bc537787f3d4d7bff(frontend/…/account/Auth.tsx::loginWithToken):::mainFlowStyle


classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9

%% Swimm:
%% graph TD;
%%       7071df105802ada928e6f2dc24a0786c5eb20f9ea01b03eec196281089daa938(<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>::<SwmToken path="frontend/src/components/account/Auth.tsx" pos="38:6:6" line-data="export default function AuthToken() {">`AuthToken`</SwmToken>) --> 7c233e049e1faaf21b52f91b1abc7ea25a4700a287ada68bc537787f3d4d7bff(<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>::<SwmToken path="frontend/src/components/account/Auth.tsx" pos="201:4:4" line-data="async function loginWithToken(token: string) {">`loginWithToken`</SwmToken>):::mainFlowStyle
%% 
%% 2ed809f7c2bc7c95aea13195a74a179ce3364cdb9d3761f99faabd4d8b99ee92(<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>::<SwmToken path="frontend/src/components/account/Auth.tsx" pos="46:3:3" line-data="  function onAuthClicked() {">`onAuthClicked`</SwmToken>) --> 7c233e049e1faaf21b52f91b1abc7ea25a4700a287ada68bc537787f3d4d7bff(<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>::<SwmToken path="frontend/src/components/account/Auth.tsx" pos="201:4:4" line-data="async function loginWithToken(token: string) {">`loginWithToken`</SwmToken>):::mainFlowStyle
%% 
%% 
%% classDef mainFlowStyle color:#000000,fill:#7CB9F4
%% classDef rootsStyle color:#000000,fill:#00FFF4
%% classDef Style1 color:#000000,fill:#00FFAA
%% classDef Style2 color:#000000,fill:#FFFF00
%% classDef Style3 color:#000000,fill:#AA7CB9
```

# Token-Based Login Entry Point

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["User provides token to log in"] --> node2{"Is system available?"}
    click node1 openCode "frontend/src/components/account/Auth.tsx:201:202"
    node2 -->|"No"| node3["Login failed: system unavailable (417)"]
    click node2 openCode "frontend/src/components/account/Auth.tsx:203:207"
    click node3 openCode "frontend/src/components/account/Auth.tsx:205:207"
    node2 -->|"Yes"| node4["Authenticate user with token"]
    click node4 openCode "frontend/src/components/account/Auth.tsx:209:210"
    node4 --> node5{"Is authentication successful?"}
    click node5 openCode "frontend/src/components/account/Auth.tsx:210:212"
    node5 -->|"Yes"| node6["Login successful (200)"]
    click node6 openCode "frontend/src/components/account/Auth.tsx:212:213"
    node5 -->|"No"| node7["Login failed: return error status from authentication"]
    click node7 openCode "frontend/src/components/account/Auth.tsx:213:216"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["User provides token to log in"] --> node2{"Is system available?"}
%%     click node1 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:201:202"
%%     node2 -->|"No"| node3["Login failed: system unavailable (417)"]
%%     click node2 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:203:207"
%%     click node3 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:205:207"
%%     node2 -->|"Yes"| node4["Authenticate user with token"]
%%     click node4 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:209:210"
%%     node4 --> node5{"Is authentication successful?"}
%%     click node5 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:210:212"
%%     node5 -->|"Yes"| node6["Login successful (200)"]
%%     click node6 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:212:213"
%%     node5 -->|"No"| node7["Login failed: return error status from authentication"]
%%     click node7 openCode "<SwmPath>[frontend/…/account/Auth.tsx](frontend/src/components/account/Auth.tsx)</SwmPath>:213:216"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/components/account/Auth.tsx" line="201">

---

LoginWithToken kicks off the login flow by checking for a cluster, setting the token for that cluster, and then immediately verifying the token's validity with <SwmToken path="frontend/src/components/account/Auth.tsx" pos="210:3:3" line-data="    await testAuth();">`testAuth`</SwmToken>. Returning 200 or 417 is just standard HTTP signaling for success or missing cluster. We call <SwmToken path="frontend/src/components/account/Auth.tsx" pos="210:3:3" line-data="    await testAuth();">`testAuth`</SwmToken> next to confirm the token works for the cluster context, not just that it was set.

```tsx
async function loginWithToken(token: string) {
  try {
    const cluster = getCluster();
    if (!cluster) {
      // Expectation failed.
      return 417;
    }

    await setToken(cluster, token);
    await testAuth();

    return 200;
  } catch (err) {
    console.error(err);
    return (err as ApiError).status;
  }
}
```

---

</SwmSnippet>

# Token Verification via Cluster API

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1{"Is cluster provided?"}
    click node1 openCode "frontend/src/lib/k8s/api/v1/clusterApi.ts:34:42"
    node1 -->|"Yes"| node2["Use provided cluster"]
    click node2 openCode "frontend/src/lib/k8s/api/v1/clusterApi.ts:34:42"
    node1 -->|"No"| node3["Use default cluster"]
    click node3 openCode "frontend/src/lib/k8s/api/v1/clusterApi.ts:34:42"
    node2 --> node4["Submit authorization review for namespace"]
    click node4 openCode "frontend/src/lib/k8s/api/v1/clusterApi.ts:38:42"
    node3 --> node4
    node4 --> node5["Return user's permissions for namespace"]
    click node5 openCode "frontend/src/lib/k8s/api/v1/clusterApi.ts:38:42"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1{"Is cluster provided?"}
%%     click node1 openCode "<SwmPath>[frontend/…/v1/clusterApi.ts](frontend/src/lib/k8s/api/v1/clusterApi.ts)</SwmPath>:34:42"
%%     node1 -->|"Yes"| node2["Use provided cluster"]
%%     click node2 openCode "<SwmPath>[frontend/…/v1/clusterApi.ts](frontend/src/lib/k8s/api/v1/clusterApi.ts)</SwmPath>:34:42"
%%     node1 -->|"No"| node3["Use default cluster"]
%%     click node3 openCode "<SwmPath>[frontend/…/v1/clusterApi.ts](frontend/src/lib/k8s/api/v1/clusterApi.ts)</SwmPath>:34:42"
%%     node2 --> node4["Submit authorization review for namespace"]
%%     click node4 openCode "<SwmPath>[frontend/…/v1/clusterApi.ts](frontend/src/lib/k8s/api/v1/clusterApi.ts)</SwmPath>:38:42"
%%     node3 --> node4
%%     node4 --> node5["Return user's permissions for namespace"]
%%     click node5 openCode "<SwmPath>[frontend/…/v1/clusterApi.ts](frontend/src/lib/k8s/api/v1/clusterApi.ts)</SwmPath>:38:42"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v1/clusterApi.ts" line="34">

---

TestAuth sends a POST to the Kubernetes selfsubjectrulesreviews endpoint to check what the token can actually do in the cluster. It uses <SwmToken path="frontend/src/lib/k8s/api/v1/clusterApi.ts" pos="36:11:11" line-data="  const clusterName = cluster || getCluster();">`getCluster`</SwmToken> to figure out which cluster to target if none is specified, and sets a 5 second timeout to avoid hanging. The next step is calling post in <SwmToken path="frontend/src/lib/k8s/api/v1/clusterApi.ts" pos="26:14:14" line-data="import type { ClusterRequest } from &#39;./clusterRequests&#39;;">`clusterRequests`</SwmToken> to actually send the request.

```typescript
export async function testAuth(cluster = '', namespace = 'default') {
  const spec = { namespace };
  const clusterName = cluster || getCluster();

  return post('/apis/authorization.k8s.io/v1/selfsubjectrulesreviews', { spec }, false, {
    timeout: 5 * 1000,
    cluster: clusterName,
  });
}
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v1/clusterRequests.ts" line="225">

---

Post wraps up the request logic by merging defaults and custom options, stringifying the payload, and picking the cluster context. It then hands everything off to <SwmToken path="frontend/src/lib/k8s/api/v1/clusterRequests.ts" pos="234:3:3" line-data="  return clusterRequest(url, {">`clusterRequest`</SwmToken>, which actually sends the HTTP POST. <SwmToken path="frontend/src/lib/k8s/api/v1/clusterRequests.ts" pos="228:1:1" line-data="  autoLogoutOnAuthError: boolean = true,">`autoLogoutOnAuthError`</SwmToken> is set to true by default, so failed auth will log the user out automatically.

```typescript
export function post(
  url: string,
  json: JSON | object | KubeObjectInterface,
  autoLogoutOnAuthError: boolean = true,
  options: ClusterRequestParams = {}
) {
  const { cluster: clusterName, ...requestOptions } = options;
  const body = JSON.stringify(json);
  const cluster = clusterName || getCluster() || '';
  return clusterRequest(url, {
    method: 'POST',
    body,
    headers: JSON_HEADERS,
    cluster,
    autoLogoutOnAuthError,
    ...requestOptions,
  });
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBdHlwZXNjcmlwdC1oZWFkbGFtcCUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="typescript-headlamp"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
