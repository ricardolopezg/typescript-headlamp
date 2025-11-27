---
title: Managing Query Parameters and UI State
---
This document describes how query parameters in the browser URL are managed to keep the UI state synchronized with navigation. The flow receives the current query parameter value and an initial state, and returns the updated value along with a setter function for future changes. By updating the URL and browser history, the flow allows users to share, bookmark, or revisit specific UI states, and ensures that resource actions are reflected in navigation.

```mermaid
flowchart TD
  node1["Managing Query Params and Resource Deletion
Read query parameter from URL
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node1 goToHeading "Managing Query Params and Resource Deletion"
  node1 --> node2{"Is initial state present and no value in URL?
(Managing Query Params and Resource Deletion)"}:::HeadingStyle
  click node2 goToHeading "Managing Query Params and Resource Deletion"
  node2 -->|"Yes"| node3["Managing Query Params and Resource Deletion
Apply initial state to URL
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node3 goToHeading "Managing Query Params and Resource Deletion"
  node2 -->|"No"| node4["Managing Query Params and Resource Deletion
User triggers value update
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node4 goToHeading "Managing Query Params and Resource Deletion"
  node4 --> node5{"Is new value undefined?
(Managing Query Params and Resource Deletion)"}:::HeadingStyle
  click node5 goToHeading "Managing Query Params and Resource Deletion"
  node5 -->|"Yes"| node6["Managing Query Params and Resource Deletion
Delete query parameter from URL
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node6 goToHeading "Managing Query Params and Resource Deletion"
  node5 -->|"No"| node7["Managing Query Params and Resource Deletion
Set query parameter in URL
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node7 goToHeading "Managing Query Params and Resource Deletion"
  node6 --> node8{"Replace or push in browser history?
(Managing Query Params and Resource Deletion)"}:::HeadingStyle
  node7 --> node8
  click node8 goToHeading "Managing Query Params and Resource Deletion"
  node8 -->|"Replace"| node9["Managing Query Params and Resource Deletion
Replace browser history
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node9 goToHeading "Managing Query Params and Resource Deletion"
  node8 -->|"Push"| node10["Managing Query Params and Resource Deletion
Push to browser history
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  click node10 goToHeading "Managing Query Params and Resource Deletion"
  node9 --> node11["Managing Query Params and Resource Deletion
Return current value and setter
(Managing Query Params and Resource Deletion)"]:::HeadingStyle
  node10 --> node11
  click node11 goToHeading "Managing Query Params and Resource Deletion"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

```mermaid
graph TD;
      cfe56ed6ea57d4da49bc07a4e2fee108921ae0742f612e29f153af538ad2e878(frontend/…/Table/Table.tsx::Table) --> aea418ebf887730e25075a63bfc85385c2d4d2b0ce74593778d152385fca1452(frontend/…/resourceMap/useQueryParamsState.tsx::useQueryParamsState):::mainFlowStyle

3e0eb8139d9a01980d464d1f7b56282926e9707ec06f0c55bdacf68730d17bc6(frontend/…/resourceMap/GraphView.tsx::GraphViewContent) --> aea418ebf887730e25075a63bfc85385c2d4d2b0ce74593778d152385fca1452(frontend/…/resourceMap/useQueryParamsState.tsx::useQueryParamsState):::mainFlowStyle


classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9

%% Swimm:
%% graph TD;
%%       cfe56ed6ea57d4da49bc07a4e2fee108921ae0742f612e29f153af538ad2e878(<SwmPath>[frontend/…/Table/Table.tsx](frontend/src/components/common/Table/Table.tsx)</SwmPath>::Table) --> aea418ebf887730e25075a63bfc85385c2d4d2b0ce74593778d152385fca1452(<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>::<SwmToken path="frontend/src/components/resourceMap/useQueryParamsState.tsx" pos="36:4:4" line-data="export function useQueryParamsState&lt;T extends string | undefined&gt;(">`useQueryParamsState`</SwmToken>):::mainFlowStyle
%% 
%% 3e0eb8139d9a01980d464d1f7b56282926e9707ec06f0c55bdacf68730d17bc6(<SwmPath>[frontend/…/resourceMap/GraphView.tsx](frontend/src/components/resourceMap/GraphView.tsx)</SwmPath>::GraphViewContent) --> aea418ebf887730e25075a63bfc85385c2d4d2b0ce74593778d152385fca1452(<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>::<SwmToken path="frontend/src/components/resourceMap/useQueryParamsState.tsx" pos="36:4:4" line-data="export function useQueryParamsState&lt;T extends string | undefined&gt;(">`useQueryParamsState`</SwmToken>):::mainFlowStyle
%% 
%% 
%% classDef mainFlowStyle color:#000000,fill:#7CB9F4
%% classDef rootsStyle color:#000000,fill:#00FFF4
%% classDef Style1 color:#000000,fill:#00FFAA
%% classDef Style2 color:#000000,fill:#FFFF00
%% classDef Style3 color:#000000,fill:#AA7CB9
```

# Managing Query Params and Resource Deletion

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Read query parameter from URL"]
  click node1 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:43:46"
  node1 --> node2{"Is initial state present and no value in URL?"}
  click node2 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:75:78"
  node2 -->|"Yes"| node3["Apply initial state to URL"]
  click node3 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:76:77"
  node2 -->|"No"| node4["User triggers value update"]
  click node4 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:49:69"
  node4 --> node5{"Is new value undefined?"}
  click node5 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:56:58"
  node5 -->|"Yes"| node6["Delete query parameter from URL"]
  click node6 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:57:58"
  node5 -->|"No"| node7["Set query parameter in URL"]
  click node7 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:59:60"
  node6 --> node8{"Replace or push in browser history?"}
  node7 --> node8
  click node8 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:64:68"
  node8 -->|"Replace"| node9["Replace browser history"]
  click node9 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:65:66"
  node8 -->|"Push"| node10["Push to browser history"]
  click node10 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:67:68"
  node9 --> node11["Return current value and setter"]
  node10 --> node11
  click node11 openCode "frontend/src/components/resourceMap/useQueryParamsState.tsx:80:81"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Read query parameter from URL"]
%%   click node1 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:43:46"
%%   node1 --> node2{"Is initial state present and no value in URL?"}
%%   click node2 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:75:78"
%%   node2 -->|"Yes"| node3["Apply initial state to URL"]
%%   click node3 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:76:77"
%%   node2 -->|"No"| node4["User triggers value update"]
%%   click node4 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:49:69"
%%   node4 --> node5{"Is new value undefined?"}
%%   click node5 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:56:58"
%%   node5 -->|"Yes"| node6["Delete query parameter from URL"]
%%   click node6 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:57:58"
%%   node5 -->|"No"| node7["Set query parameter in URL"]
%%   click node7 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:59:60"
%%   node6 --> node8{"Replace or push in browser history?"}
%%   node7 --> node8
%%   click node8 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:64:68"
%%   node8 -->|"Replace"| node9["Replace browser history"]
%%   click node9 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:65:66"
%%   node8 -->|"Push"| node10["Push to browser history"]
%%   click node10 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:67:68"
%%   node9 --> node11["Return current value and setter"]
%%   node10 --> node11
%%   click node11 openCode "<SwmPath>[frontend/…/resourceMap/useQueryParamsState.tsx](frontend/src/components/resourceMap/useQueryParamsState.tsx)</SwmPath>:80:81"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/components/resourceMap/useQueryParamsState.tsx" line="36">

---

In <SwmToken path="frontend/src/components/resourceMap/useQueryParamsState.tsx" pos="36:4:4" line-data="export function useQueryParamsState&lt;T extends string | undefined&gt;(">`useQueryParamsState`</SwmToken>, we start by reading and updating query parameters from the URL, so that state changes are reflected in navigation. We need to call into <SwmToken path="frontend/src/lib/k8s/KubeObject.ts" pos="46:4:4" line-data="export class KubeObject&lt;T extends KubeObjectInterface | KubeEvent = any&gt; {">`KubeObject`</SwmToken> next because after updating the query params, we often need to trigger resource actions (like deletion) that depend on the current state reflected in the URL.

```tsx
export function useQueryParamsState<T extends string | undefined>(
  param: string,
  initialState: T
): UseQueryParamsStateReturnType<T> {
  const { search } = useLocation();
  const history = useHistory();

  const value = useMemo(() => {
    const params = new URLSearchParams(search);
    return (params.get(param) ?? undefined) as T | undefined;
  }, [search, param]);

  const setValue = useCallback(
    (newValue: T | undefined, params: { replace?: boolean } = {}) => {
      if (newValue !== undefined && typeof newValue !== 'string') {
        throw new Error("useQueryParamsState: Can't set a value to something that isn't a string");
      }

      // Create new search params
      const newParams = new URLSearchParams(history.location.search);
      if (newValue === undefined) {
        newParams.delete(param);
      } else {
        newParams.set(param, newValue);
      }

```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/KubeObject.ts" line="450">

---

<SwmToken path="frontend/src/lib/k8s/KubeObject.ts" pos="450:1:1" line-data="  delete(force?: boolean) {">`delete`</SwmToken> in <SwmToken path="frontend/src/lib/k8s/KubeObject.ts" pos="46:4:4" line-data="export class KubeObject&lt;T extends KubeObjectInterface | KubeEvent = any&gt; {">`KubeObject`</SwmToken> builds the API call for resource deletion, handling namespaced resources by prepending the namespace to the arguments, and using <SwmToken path="frontend/src/lib/k8s/KubeObject.ts" pos="459:3:3" line-data="      params.gracePeriodSeconds = 0;">`gracePeriodSeconds`</SwmToken>=0 for force deletes. This matches backend expectations for argument order and immediate deletion.

```typescript
  delete(force?: boolean) {
    const args: string[] = [this.getName()];
    if (this.isNamespaced) {
      args.unshift(this.getNamespace()!);
    }
    const params: DeleteParameters = {};

    console.log(force);
    if (force) {
      params.gracePeriodSeconds = 0;
      console.log(params);
    }

    // @ts-ignore
    return this._class().apiEndpoint.delete(...args, params, this._clusterName);
  }
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/components/resourceMap/useQueryParamsState.tsx" line="62">

---

After returning from KubeObject.delete, <SwmToken path="frontend/src/components/resourceMap/useQueryParamsState.tsx" pos="36:4:4" line-data="export function useQueryParamsState&lt;T extends string | undefined&gt;(">`useQueryParamsState`</SwmToken> updates the URL to reflect the new state, and applies <SwmToken path="frontend/src/components/resourceMap/useQueryParamsState.tsx" pos="73:5:5" line-data="  // Apply initialState if any">`initialState`</SwmToken> if needed. This keeps the UI and navigation consistent after resource actions.

```tsx
      // Apply new search params
      const newSearch = '?' + newParams;
      if (params.replace) {
        history.replace(newSearch);
      } else {
        history.push(newSearch);
      }
    },
    [history.location.search, param]
  );

  // Apply initialState if any
  useEffect(() => {
    if (initialState && !value) {
      setValue(initialState, { replace: true });
    }
  }, [initialState]);

  return [value, setValue];
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBdHlwZXNjcmlwdC1oZWFkbGFtcCUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="typescript-headlamp"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
