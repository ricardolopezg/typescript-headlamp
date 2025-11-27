---
title: Managing Port Forward Sessions
---
This document describes how users manage port forwarding sessions by starting, stopping, or deleting them from the UI. When an action is selected, the system processes the request and updates the port forward list, ensuring the UI always displays the latest information by merging backend and local storage data.

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

```mermaid
graph TD;
      e14c25aebd7f290e896bc9310a73d2523213cf0b7b384545c570b832bf8ed132(frontend/…/portforward/index.tsx::PortForwardContextMenu) --> e2f8aae1213210b2be5c2ba54d95edd209004d85814f1487a6b58b2a37375ec1(frontend/…/portforward/index.tsx::handleAction):::mainFlowStyle

421fec0cd727bf2b617b2ac89e8ca15ea547ad1ffbfa1c7371bb6b64cce497bf(frontend/…/portforward/index.tsx::PortForwardingList) --> e2f8aae1213210b2be5c2ba54d95edd209004d85814f1487a6b58b2a37375ec1(frontend/…/portforward/index.tsx::handleAction):::mainFlowStyle


classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9

%% Swimm:
%% graph TD;
%%       e14c25aebd7f290e896bc9310a73d2523213cf0b7b384545c570b832bf8ed132(<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>::<SwmToken path="frontend/src/components/portforward/index.tsx" pos="167:3:3" line-data="  const PortForwardContextMenu = ({ portforward }: { portforward: any }) =&gt; {">`PortForwardContextMenu`</SwmToken>) --> e2f8aae1213210b2be5c2ba54d95edd209004d85814f1487a6b58b2a37375ec1(<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>::<SwmToken path="frontend/src/components/portforward/index.tsx" pos="120:3:3" line-data="  const handleAction = (option: string, portforward: any, closeMenu: () =&gt; void) =&gt; {">`handleAction`</SwmToken>):::mainFlowStyle
%% 
%% 421fec0cd727bf2b617b2ac89e8ca15ea547ad1ffbfa1c7371bb6b64cce497bf(<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>::<SwmToken path="frontend/src/components/portforward/index.tsx" pos="52:6:6" line-data="export default function PortForwardingList() {">`PortForwardingList`</SwmToken>) --> e2f8aae1213210b2be5c2ba54d95edd209004d85814f1487a6b58b2a37375ec1(<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>::<SwmToken path="frontend/src/components/portforward/index.tsx" pos="120:3:3" line-data="  const handleAction = (option: string, portforward: any, closeMenu: () =&gt; void) =&gt; {">`handleAction`</SwmToken>):::mainFlowStyle
%% 
%% 
%% classDef mainFlowStyle color:#000000,fill:#7CB9F4
%% classDef rootsStyle color:#000000,fill:#00FFF4
%% classDef Style1 color:#000000,fill:#00FFAA
%% classDef Style2 color:#000000,fill:#FFFF00
%% classDef Style3 color:#000000,fill:#AA7CB9
```

# Handling Port Forward Actions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Close menu"] --> node2{"Is option valid?"}
  click node1 openCode "frontend/src/components/portforward/index.tsx:121:121"
  click node2 openCode "frontend/src/components/portforward/index.tsx:122:124"
  node2 -->|"No"| node3["Return"]
  click node3 openCode "frontend/src/components/portforward/index.tsx:123:123"
  node2 -->|"Yes"| node4{"Which action?"}
  click node4 openCode "frontend/src/components/portforward/index.tsx:128:128"
  node4 -->|"Start"| node5{"Is Docker Desktop?"}
  click node5 openCode "frontend/src/components/portforward/index.tsx:130:132"
  node5 -->|"Yes"| node6["Set address to 0.0.0.0"]
  click node6 openCode "frontend/src/components/portforward/index.tsx:131:132"
  node5 -->|"No"| node7["Set address to localhost"]
  click node7 openCode "frontend/src/components/portforward/index.tsx:129:129"
  node6 --> node8["Prepare and open start dialog"]
  click node8 openCode "frontend/src/components/portforward/index.tsx:133:134"
  node7 --> node8
  node8 --> node17["Return"]
  click node17 openCode "frontend/src/components/portforward/index.tsx:135:136"
  node4 -->|"Stop"| node9["Mark port forward as in action"]
  click node9 openCode "frontend/src/components/portforward/index.tsx:138:139"
  node9 --> node10["Stop port forward and update list"]
  click node10 openCode "frontend/src/components/portforward/index.tsx:140:144"
  node4 -->|"Delete"| node11["Mark port forward as in action"]
  click node11 openCode "frontend/src/components/portforward/index.tsx:147:148"
  node11 --> node12["Delete port forward"]
  click node12 openCode "frontend/src/components/portforward/index.tsx:149:163"
  node12 --> node13{"Exists in storage?"}
  click node13 openCode "frontend/src/components/portforward/index.tsx:153:155"
  node13 -->|"Yes"| node14["Remove from storage"]
  click node14 openCode "frontend/src/components/portforward/index.tsx:157:159"
  node13 -->|"No"| node15["Skip removal"]
  click node15 openCode "frontend/src/components/portforward/index.tsx:156:156"
  node14 --> node16["Update port forward list"]
  click node16 openCode "frontend/src/components/portforward/index.tsx:162:163"
  node15 --> node16
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Close menu"] --> node2{"Is option valid?"}
%%   click node1 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:121:121"
%%   click node2 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:122:124"
%%   node2 -->|"No"| node3["Return"]
%%   click node3 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:123:123"
%%   node2 -->|"Yes"| node4{"Which action?"}
%%   click node4 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:128:128"
%%   node4 -->|"Start"| node5{"Is Docker Desktop?"}
%%   click node5 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:130:132"
%%   node5 -->|"Yes"| node6["Set address to <SwmToken path="frontend/src/components/portforward/index.tsx" pos="131:6:12" line-data="        address = &#39;0.0.0.0&#39;;">`0.0.0.0`</SwmToken>"]
%%   click node6 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:131:132"
%%   node5 -->|"No"| node7["Set address to localhost"]
%%   click node7 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:129:129"
%%   node6 --> node8["Prepare and open start dialog"]
%%   click node8 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:133:134"
%%   node7 --> node8
%%   node8 --> node17["Return"]
%%   click node17 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:135:136"
%%   node4 -->|"Stop"| node9["Mark port forward as in action"]
%%   click node9 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:138:139"
%%   node9 --> node10["Stop port forward and update list"]
%%   click node10 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:140:144"
%%   node4 -->|"Delete"| node11["Mark port forward as in action"]
%%   click node11 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:147:148"
%%   node11 --> node12["Delete port forward"]
%%   click node12 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:149:163"
%%   node12 --> node13{"Exists in storage?"}
%%   click node13 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:153:155"
%%   node13 -->|"Yes"| node14["Remove from storage"]
%%   click node14 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:157:159"
%%   node13 -->|"No"| node15["Skip removal"]
%%   click node15 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:156:156"
%%   node14 --> node16["Update port forward list"]
%%   click node16 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:162:163"
%%   node15 --> node16
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/components/portforward/index.tsx" line="120">

---

HandleAction kicks off the flow by branching on the selected action (Start, Stop, Delete) for a port forward. It expects portforward to have id, namespace, and cluster, but doesn't enforce this in the signature. For Start, it sets up the address (using Docker Desktop detection) and opens a dialog. For Stop and Delete, it sets a loading state, calls <SwmToken path="frontend/src/components/portforward/index.tsx" pos="140:1:1" line-data="      stopOrDeletePortForward(cluster, id, true).finally(() =&gt; {">`stopOrDeletePortForward`</SwmToken>, and then fetches the updated port forward list to sync the UI. Delete also removes the entry from <SwmToken path="frontend/src/components/portforward/index.tsx" pos="153:7:7" line-data="        const portforwardInStorage = localStorage.getItem(PORT_FORWARDS_STORAGE_KEY);">`localStorage`</SwmToken>. The call to <SwmToken path="frontend/src/components/portforward/index.tsx" pos="143:1:1" line-data="        fetchPortForwardList(true);">`fetchPortForwardList`</SwmToken> is what keeps the UI in sync after these changes.

```tsx
  const handleAction = (option: string, portforward: any, closeMenu: () => void) => {
    closeMenu();
    if (!option || typeof option !== 'string') {
      return;
    }

    const { id, namespace, cluster } = portforward;

    if (option === PortForwardAction.Start) {
      let address = 'localhost';
      if (isDockerDesktop()) {
        address = '0.0.0.0';
      }
      setSelectedForStart({ ...portforward, cluster, namespace, address });
      setStartDialogOpen(true);
      return;
    }
    if (option === PortForwardAction.Stop) {
      setPortForwardInAction({ ...portforward, loading: true });
      // stop portforward
      stopOrDeletePortForward(cluster, id, true).finally(() => {
        setPortForwardInAction(null);
        // update portforward list item
        fetchPortForwardList(true);
      });
    }
    if (option === PortForwardAction.Delete) {
      setPortForwardInAction({ ...portforward, loading: true });
      // delete portforward
      stopOrDeletePortForward(cluster, id, false).finally(() => {
        setPortForwardInAction(null);

        // remove portforward from storage too
        const portforwardInStorage = localStorage.getItem(PORT_FORWARDS_STORAGE_KEY);
        const parsedPortForwards = JSON.parse(portforwardInStorage || '[]');
        const index = parsedPortForwards.findIndex((pf: any) => pf.id === id);
        if (index !== -1) {
          parsedPortForwards.splice(index, 1);
        }
        localStorage.setItem(PORT_FORWARDS_STORAGE_KEY, JSON.stringify(parsedPortForwards));

        // update portforward list item
        fetchPortForwardList(true);
      });
    }
  };
```

---

</SwmSnippet>

# Fetching and Syncing Port Forward List

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Is cluster selected?"}
  click node1 openCode "frontend/src/components/portforward/index.tsx:71:72"
  node1 -->|"Yes"| node2["Fetch port forward list from backend"]
  click node2 openCode "frontend/src/components/portforward/index.tsx:75:75"
  node1 -->|"No"| node16["End"]
  click node16 openCode "frontend/src/components/portforward/index.tsx:72:72"
  node2 --> node3["Prepare port forward list"]
  click node3 openCode "frontend/src/components/portforward/index.tsx:76:76"

  subgraph loop1["For each backend port forward"]
    node3 --> node4{"Is port forward in action with error and showError?"}
    click node4 openCode "frontend/src/components/portforward/index.tsx:78:79"
    node4 -->|"Yes"| node5["Display error notification to user"]
    click node5 openCode "frontend/src/components/portforward/index.tsx:80:85"
    node5 --> node6["Continue"]
    click node6 openCode "frontend/src/components/portforward/index.tsx:86:87"
    node4 -->|"No"| node6
  end
  node6 --> node8["Sync with local storage"]
  click node8 openCode "frontend/src/components/portforward/index.tsx:91:92"

  subgraph loop2["For each local port forward"]
    node8 --> node9{"Is local port forward missing from backend list?"}
    click node9 openCode "frontend/src/components/portforward/index.tsx:94:95"
    node9 -->|"Yes"| node10["Add to list with status 'stop'"]
    click node10 openCode "frontend/src/components/portforward/index.tsx:96:98"
    node9 -->|"No"| node11["Continue"]
    click node11 openCode "frontend/src/components/portforward/index.tsx:99:99"
  end
  node11 --> node12["Update status of all port forwards to 'stop'"]
  click node12 openCode "frontend/src/components/portforward/index.tsx:106:110"

  subgraph loop3["For each port forward"]
    node12 --> node13["Set status to 'stop' before saving"]
    click node13 openCode "frontend/src/components/portforward/index.tsx:107:109"
  end
  node13 --> node14["Save to local storage"]
  click node14 openCode "frontend/src/components/portforward/index.tsx:100:112"
  node14 --> node15["Update UI with final list"]
  click node15 openCode "frontend/src/components/portforward/index.tsx:113:113"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Is cluster selected?"}
%%   click node1 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:71:72"
%%   node1 -->|"Yes"| node2["Fetch port forward list from backend"]
%%   click node2 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:75:75"
%%   node1 -->|"No"| node16["End"]
%%   click node16 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:72:72"
%%   node2 --> node3["Prepare port forward list"]
%%   click node3 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:76:76"
%% 
%%   subgraph loop1["For each backend port forward"]
%%     node3 --> node4{"Is port forward in action with error and <SwmToken path="frontend/src/components/portforward/index.tsx" pos="70:5:5" line-data="  function fetchPortForwardList(showError?: boolean) {">`showError`</SwmToken>?"}
%%     click node4 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:78:79"
%%     node4 -->|"Yes"| node5["Display error notification to user"]
%%     click node5 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:80:85"
%%     node5 --> node6["Continue"]
%%     click node6 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:86:87"
%%     node4 -->|"No"| node6
%%   end
%%   node6 --> node8["Sync with local storage"]
%%   click node8 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:91:92"
%% 
%%   subgraph loop2["For each local port forward"]
%%     node8 --> node9{"Is local port forward missing from backend list?"}
%%     click node9 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:94:95"
%%     node9 -->|"Yes"| node10["Add to list with status 'stop'"]
%%     click node10 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:96:98"
%%     node9 -->|"No"| node11["Continue"]
%%     click node11 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:99:99"
%%   end
%%   node11 --> node12["Update status of all port forwards to 'stop'"]
%%   click node12 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:106:110"
%% 
%%   subgraph loop3["For each port forward"]
%%     node12 --> node13["Set status to 'stop' before saving"]
%%     click node13 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:107:109"
%%   end
%%   node13 --> node14["Save to local storage"]
%%   click node14 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:100:112"
%%   node14 --> node15["Update UI with final list"]
%%   click node15 openCode "<SwmPath>[frontend/…/portforward/index.tsx](frontend/src/components/portforward/index.tsx)</SwmPath>:113:113"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/components/portforward/index.tsx" line="70">

---

In <SwmToken path="frontend/src/components/portforward/index.tsx" pos="70:3:3" line-data="  function fetchPortForwardList(showError?: boolean) {">`fetchPortForwardList`</SwmToken>, we start by checking for a cluster and then call <SwmToken path="frontend/src/components/portforward/index.tsx" pos="75:1:1" line-data="    listPortForward(cluster).then(portforwards =&gt; {">`listPortForward`</SwmToken> to get the latest port forwards from the backend. This is necessary because <SwmToken path="frontend/src/components/portforward/index.tsx" pos="90:13:13" line-data="      // sync portforwards from backend with localStorage">`localStorage`</SwmToken> only keeps track of port forwards for persistence, but the backend has the real-time status. The next step is to merge and sync these two sources.

```tsx
  function fetchPortForwardList(showError?: boolean) {
    const cluster = getCluster();
    if (!cluster) return;

    // fetch port forwarding list
    listPortForward(cluster).then(portforwards => {
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v1/portForward.ts" line="175">

---

ListPortForward fetches the port forward list for a cluster by building headers (including a user ID if the cluster is dynamic) and calling <SwmToken path="frontend/src/lib/k8s/api/v1/portForward.ts" pos="184:3:3" line-data="  return clusterFetch(`/portforward/list`, {">`clusterFetch`</SwmToken> to hit the backend endpoint. The result is a promise with the port forward data.

```typescript
export async function listPortForward(cluster: string): Promise<PortForward[]> {
  const kubeconfig = await findKubeconfigByClusterName(cluster);
  const headers = new Headers(addBackstageAuthHeaders(JSON_HEADERS));

  // This means cluster is dynamically configured.
  if (kubeconfig !== null) {
    headers.set('X-HEADLAMP-USER-ID', getUserIdFromLocalStorage());
  }

  return clusterFetch(`/portforward/list`, {
    headers: headers,
    cluster,
  }).then(response => response.json());
}
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/components/portforward/index.tsx" line="76">

---

We just got the port forward list from <SwmToken path="frontend/src/components/portforward/index.tsx" pos="75:1:1" line-data="    listPortForward(cluster).then(portforwards =&gt; {">`listPortForward`</SwmToken>. Now, <SwmToken path="frontend/src/components/portforward/index.tsx" pos="70:3:3" line-data="  function fetchPortForwardList(showError?: boolean) {">`fetchPortForwardList`</SwmToken> checks for errors, merges backend and <SwmToken path="frontend/src/components/portforward/index.tsx" pos="90:13:13" line-data="      // sync portforwards from backend with localStorage">`localStorage`</SwmToken> port forwards (adding missing ones with stop status), updates <SwmToken path="frontend/src/components/portforward/index.tsx" pos="90:13:13" line-data="      // sync portforwards from backend with localStorage">`localStorage`</SwmToken> so everything is marked as stopped, and sets the state for the UI. This keeps things persistent but always trusts the backend for live status.

```tsx
      const massagedPortForwards = portforwards === null ? [] : portforwards;
      massagedPortForwards.forEach((portforward: any) => {
        if (portForwardInAction?.id === portforward.id) {
          if (portforward.Error && showError) {
            enqueueSnackbar(portforward.Error, {
              key: 'portforward-error',
              preventDuplicate: true,
              autoHideDuration: 3000,
              variant: 'error',
            });
          }
        }
      });

      // sync portforwards from backend with localStorage
      const portforwardInStorage = localStorage.getItem(PORT_FORWARDS_STORAGE_KEY);
      const parsedPortForwards = JSON.parse(portforwardInStorage || '[]');
      parsedPortForwards.forEach((portforward: any) => {
        const index = massagedPortForwards.findIndex((pf: any) => pf.id === portforward.id);
        if (index === -1) {
          portforward.status = PORT_FORWARD_STOP_STATUS;
          massagedPortForwards.push(portforward);
        }
      });
      localStorage.setItem(
        PORT_FORWARDS_STORAGE_KEY,
        JSON.stringify(
          // in the locaStorage we store portforward status as stop
          // this is because the correct status is always present on the backend
          // the localStorage portforwards are used specifically when the user relaunches the app
          massagedPortForwards.map((portforward: any) => {
            const newPortforward = { ...portforward };
            newPortforward.status = PORT_FORWARD_STOP_STATUS;
            return newPortforward;
          })
        )
      );
      setPortForwards(massagedPortForwards);
    });
  }
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBdHlwZXNjcmlwdC1oZWFkbGFtcCUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="typescript-headlamp"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
