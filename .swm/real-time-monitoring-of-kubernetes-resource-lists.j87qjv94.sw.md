---
title: Real-time monitoring of Kubernetes resource lists
---
This document describes how the system enables real-time monitoring of multiple Kubernetes resource lists, updating the UI as changes occur. The process starts by choosing between multiplexed or legacy <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections. Multiplexed connections efficiently monitor many resources over fewer sockets, while legacy connections use a separate socket for each resource list. As updates arrive, the UI data is refreshed to reflect the latest cluster and namespace information.

```mermaid
flowchart TD
  node1["Deciding WebSocket Strategy"]:::HeadingStyle
  click node1 goToHeading "Deciding WebSocket Strategy"
  node1 -->|"Multiplexed enabled"| node2["Setting Up Multiplexed WebSocket Connections"]:::HeadingStyle
  click node2 goToHeading "Setting Up Multiplexed WebSocket Connections"
  node1 -->|"Legacy"| node3["Setting Up Legacy WebSocket Connections"]:::HeadingStyle
  click node3 goToHeading "Setting Up Legacy WebSocket Connections"
  node2 --> node4["Managing Multiple WebSocket Connections"]:::HeadingStyle
  click node4 goToHeading "Managing Multiple WebSocket Connections"
  node3 --> node4
  node4 --> node5["Processing Incoming WebSocket Messages"]:::HeadingStyle
  click node5 goToHeading "Processing Incoming WebSocket Messages"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% flowchart TD
%%   node1["Deciding <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Strategy"]:::HeadingStyle
%%   click node1 goToHeading "Deciding <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Strategy"
%%   node1 -->|"Multiplexed enabled"| node2["Setting Up Multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"]:::HeadingStyle
%%   click node2 goToHeading "Setting Up Multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%%   node1 -->|"Legacy"| node3["Setting Up Legacy <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"]:::HeadingStyle
%%   click node3 goToHeading "Setting Up Legacy <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%%   node2 --> node4["Managing Multiple <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"]:::HeadingStyle
%%   click node4 goToHeading "Managing Multiple <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%%   node3 --> node4
%%   node4 --> node5["Processing Incoming <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Messages"]:::HeadingStyle
%%   click node5 goToHeading "Processing Incoming <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Messages"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Deciding <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Strategy

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="127">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="127:4:4" line-data="export function useWatchKubeObjectLists&lt;K extends KubeObject&gt;({">`useWatchKubeObjectLists`</SwmToken>, we start by checking if multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections are enabled. If so, we delegate to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, which lets us handle multiple resource lists over fewer connections. This is necessary to optimize resource usage and avoid opening a ton of sockets when watching lots of clusters/namespaces.

```typescript
export function useWatchKubeObjectLists<K extends KubeObject>({
  kubeObjectClass,
  endpoint,
  lists,
  queryParams,
}: {
  /** KubeObject class of the watched resource list */
  kubeObjectClass: (new (...args: any) => K) & typeof KubeObject<any>;
  /** Query parameters for the WebSocket connection URL */
  queryParams?: QueryParameters;
  /** Kube resource API endpoint information */
  endpoint?: KubeObjectEndpoint | null;
  /** Which clusters and namespaces to watch */
  lists: Array<{ cluster: string; namespace?: string; resourceVersion: string }>;
}) {
  if (getWebsocketMultiplexerEnabled()) {
    return useWatchKubeObjectListsMultiplexed({
      kubeObjectClass,
      endpoint,
      lists,
      queryParams,
    });
  } else {
```

---

</SwmSnippet>

## Setting Up Multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Initialize connections and parameters"]
    click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:170:215"
    node1 --> node2{"Are endpoint and connections defined?"}
    click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:261:263"
    node2 -->|"Yes"| node3["Set up subscriptions for each connection"]
    node2 -->|"No"| node4["Finalizing Multiplexed WebSocket Subscriptions"]
    click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:268:285"
    

    subgraph loop1["For each connection"]
      node3 --> node5["Subscribe to updates"]
      click node5 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:268:285"
      node5 --> node6{"Did subscription fail and retryCount < 3?"}
      click node6 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:279:283"
      node6 -->|"Yes"| node5
      node6 -->|"No"| node4
      node5 --> node7["Process incoming update"]
      click node7 openCode "app/electron/main.ts:365:370"
      node7 --> node3
    end

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node4 goToHeading "Finalizing Multiplexed WebSocket Subscriptions"
node4:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Initialize connections and parameters"]
%%     click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:170:215"
%%     node1 --> node2{"Are endpoint and connections defined?"}
%%     click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:261:263"
%%     node2 -->|"Yes"| node3["Set up subscriptions for each connection"]
%%     node2 -->|"No"| node4["Finalizing Multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Subscriptions"]
%%     click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:268:285"
%%     
%% 
%%     subgraph loop1["For each connection"]
%%       node3 --> node5["Subscribe to updates"]
%%       click node5 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:268:285"
%%       node5 --> node6{"Did subscription fail and <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="278:3:3" line-data="          const retryCount = parseInt(parsedUrl.searchParams.get(&#39;retryCount&#39;) || &#39;0&#39;);">`retryCount`</SwmToken> < 3?"}
%%       click node6 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:279:283"
%%       node6 -->|"Yes"| node5
%%       node6 -->|"No"| node4
%%       node5 --> node7["Process incoming update"]
%%       click node7 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:365:370"
%%       node7 --> node3
%%     end
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node4 goToHeading "Finalizing Multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Subscriptions"
%% node4:::HeadingStyle
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="170">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="170:2:2" line-data="function useWatchKubeObjectListsMultiplexed&lt;K extends KubeObject&gt;({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, we prep the resource version tracking and build stable connection <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="190:9:9" line-data="  // Create stable connection URLs for each list">`URLs`</SwmToken> for each watched list. We also set up the update handler for incoming <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="203:5:5" line-data="      // Construct WebSocket URL with current parameters">`WebSocket`</SwmToken> messages. Next, we need to call into the Electron main process to handle plugin updates, since that's where the actual update logic and cache management happens.

```typescript
function useWatchKubeObjectListsMultiplexed<K extends KubeObject>({
  kubeObjectClass,
  endpoint,
  lists,
  queryParams,
}: {
  kubeObjectClass: (new (...args: any) => K) & typeof KubeObject<any>;
  endpoint?: KubeObjectEndpoint | null;
  lists: Array<{ cluster: string; namespace?: string; resourceVersion: string }>;
  queryParams?: QueryParameters;
}): void {
  const client = useQueryClient();

  // Track the latest resource versions to prevent duplicate updates
  const latestResourceVersions = useRef<Record<string, string>>({});

  // Stabilize queryParams to prevent unnecessary effect triggers
  // Only update when the stringified params change
  const stableQueryParams = useMemo(() => queryParams, [JSON.stringify(queryParams)]);

  // Create stable connection URLs for each list
  // Updates only when endpoint, lists, or stableQueryParams change
  const connections = useMemo(() => {
    if (!endpoint) {
      return [];
    }

    return lists.map(list => {
      const key = `${list.cluster}:${list.namespace || ''}`;

      // Always use the latest resource version from the server
      latestResourceVersions.current[key] = list.resourceVersion;

      // Construct WebSocket URL with current parameters
      return {
        url: makeUrl([KubeObjectEndpoint.toUrl(endpoint, list.namespace)], {
          ...stableQueryParams,
          watch: 1,
          resourceVersion: latestResourceVersions.current[key],
        }),
        cluster: list.cluster,
        namespace: list.namespace,
      };
    });
  }, [endpoint, lists, stableQueryParams]);

  // Create stable update handler to process WebSocket messages
  // Re-create only when dependencies change
  const handleUpdate = useCallback(
    (update: any, cluster: string, namespace: string | undefined) => {
      if (!update || typeof update !== 'object' || !endpoint) {
        return;
      }

      const key = `${cluster}:${namespace || ''}`;

      // Update resource version from incoming message
      if (update.object?.metadata?.resourceVersion) {
        latestResourceVersions.current[key] = update.object.metadata.resourceVersion;
      }

      // Create query key for React Query cache
      const queryKey = kubeObjectListQuery<K>(
        kubeObjectClass,
        endpoint,
        namespace,
        cluster,
        stableQueryParams ?? {}
      ).queryKey;

      // Update React Query cache with new data
      client.setQueryData(queryKey, (oldResponse: ListResponse<any> | undefined | null) => {
        if (!oldResponse) {
          return oldResponse;
        }

        const newList = KubeList.applyUpdate(oldResponse.list, update, kubeObjectClass, cluster);

        // Only update if the list actually changed
        if (newList === oldResponse.list) {
          return oldResponse;
        }

        return { ...oldResponse, list: newList };
      });
    },
    [client, kubeObjectClass, endpoint, stableQueryParams]
  );

  // Set up WebSocket subscriptions
  useEffect(() => {
    if (!endpoint || connections.length === 0) {
      return;
    }

    const cleanups: (() => void)[] = [];

    // Create subscriptions for each connection
    connections.forEach(({ url, cluster, namespace }) => {
      const parsedUrl = new URL(url, BASE_WS_URL);

      // Subscribe to WebSocket updates
      WebSocketManager.subscribe(cluster, parsedUrl.pathname, parsedUrl.search.slice(1), update =>
        handleUpdate(update, cluster, namespace)
      ).then(
        cleanup => cleanups.push(cleanup),
        error => {
```

---

</SwmSnippet>

### Handling Plugin Update Requests

<SwmSnippet path="/app/electron/main.ts" line="365">

---

<SwmToken path="app/electron/main.ts" pos="365:3:3" line-data="  private handleUpdate(eventData: Action, updateCache: (progress: ProgressResp) =&gt; void) {">`handleUpdate`</SwmToken> checks for a valid plugin name, sets up progress tracking, and then calls <SwmToken path="app/electron/main.ts" pos="383:1:3" line-data="    PluginManager.update(">`PluginManager.update`</SwmToken> to actually perform the update. This hands off the heavy lifting to the plugin management logic, which knows how to fetch, compare, and replace plugin files.

```typescript
  private handleUpdate(eventData: Action, updateCache: (progress: ProgressResp) => void) {
    const { identifier, pluginName, destinationFolder, headlampVersion } = eventData;
    if (!pluginName) {
      this.cache[identifier] = {
        action: 'UPDATE',
        progress: { type: 'error', message: 'Plugin Name is required' },
      };
      return;
    }

    const controller = new AbortController();
    this.cache[identifier] = {
      action: 'UPDATE',
      percentage: 10,
      progress: { type: 'info', message: 'updating plugin' },
      controller,
    };

    PluginManager.update(
      pluginName,
      destinationFolder,
      headlampVersion,
      progress => {
        updateCache(progress);
      },
      controller.signal
    );
  }
```

---

</SwmSnippet>

### Validating and Downloading Plugin Updates

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start plugin update process"]
  click node1 openCode "app/electron/plugin-management.ts:239:246"
  node1 --> node2{"Is plugin installed?"}
  click node2 openCode "app/electron/plugin-management.ts:248:251"
  node2 -->|"No"| node3["Notify user: Plugin not installed"]
  click node3 openCode "app/electron/plugin-management.ts:250:251"
  node2 -->|"Yes"| node4{"Is plugin found?"}
  click node4 openCode "app/electron/plugin-management.ts:252:255"
  node4 -->|"No"| node5["Notify user: Plugin not found"]
  click node5 openCode "app/electron/plugin-management.ts:254:255"
  node4 -->|"Yes"| node6{"Is newer version available?"}
  click node6 openCode "app/electron/plugin-management.ts:267:269"
  node6 -->|"No"| node7["Notify user: No updates available"]
  click node7 openCode "app/electron/plugin-management.ts:268:269"
  node6 -->|"Yes"| node8["Download, check compatibility, and update plugin"]
  click node8 openCode "app/electron/plugin-management.ts:272:293"
  node8 --> node9["Notify user: Plugin updated successfully"]
  click node9 openCode "app/electron/plugin-management.ts:294:296"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start plugin update process"]
%%   click node1 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:239:246"
%%   node1 --> node2{"Is plugin installed?"}
%%   click node2 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:248:251"
%%   node2 -->|"No"| node3["Notify user: Plugin not installed"]
%%   click node3 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:250:251"
%%   node2 -->|"Yes"| node4{"Is plugin found?"}
%%   click node4 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:252:255"
%%   node4 -->|"No"| node5["Notify user: Plugin not found"]
%%   click node5 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:254:255"
%%   node4 -->|"Yes"| node6{"Is newer version available?"}
%%   click node6 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:267:269"
%%   node6 -->|"No"| node7["Notify user: No updates available"]
%%   click node7 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:268:269"
%%   node6 -->|"Yes"| node8["Download, check compatibility, and update plugin"]
%%   click node8 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:272:293"
%%   node8 --> node9["Notify user: Plugin updated successfully"]
%%   click node9 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:294:296"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/app/electron/plugin-management.ts" line="239">

---

We check if the plugin needs updating, download and extract the new version, and prep for replacing the old files.

```typescript
  static async update(
    pluginName: string,
    destinationFolder: string = defaultUserPluginsDir(),
    headlampVersion: string = '',
    progressCallback: null | ProgressCallback = null,
    signal: AbortSignal | null = null
  ): Promise<void> {
    try {
      // @todo: should list call take progressCallback?
      const installedPlugins = PluginManager.list(destinationFolder);
      if (!installedPlugins) {
        throw new Error('InstalledPlugins not found');
      }
      const plugin = installedPlugins.find(p => p.pluginName === pluginName);
      if (!plugin) {
        throw new Error('Plugin not found');
      }

      const pluginDir = path.join(destinationFolder, plugin.folderName);
      // read the package.json of the plugin
      const packageJsonPath = path.join(pluginDir, 'package.json');
      const packageJson = JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));

      const pluginData = await fetchPluginInfo(plugin.artifacthubURL, progressCallback, signal);

      const latestVersion = pluginData.version;
      const currentVersion = packageJson.artifacthub.version;

      if (semver.lte(latestVersion, currentVersion)) {
        throw new Error('No updates available');
      }

      // eslint-disable-next-line no-unused-vars
      const [_, tempFolder] = await downloadExtractArchive(
        pluginData,
        headlampVersion,
        progressCallback,
        signal
      );

```

---

</SwmSnippet>

<SwmSnippet path="/app/electron/plugin-management.ts" line="469">

---

<SwmToken path="app/electron/plugin-management.ts" pos="469:4:4" line-data="async function downloadExtractArchive(">`downloadExtractArchive`</SwmToken> checks plugin compatibility, creates a temp directory, downloads and extracts the plugin archive and extra files, and then adds artifacthub metadata and a management flag to the extracted <SwmPath>[package.json](package.json)</SwmPath>. Progress and cancellation are handled via callbacks and signals.

```typescript
async function downloadExtractArchive(
  pluginInfo: ArtifactHubHeadlampPkg,
  headlampVersion: string,
  progressCallback: ProgressCallback | null,
  signal: AbortSignal | null
): Promise<[string, string]> {
  // fetch plugin metadata
  if (signal && signal.aborted) {
    throw new Error('Download cancelled');
  }

  const pluginName = pluginInfo.name;
  if (!validatePluginName(pluginName)) {
    throw new Error('Invalid plugin name');
  }

  // Check if the plugin is compatible with the current Headlamp version
  if (headlampVersion) {
    if (progressCallback) {
      progressCallback({ type: 'info', message: 'Checking compatibility with Headlamp version' });
    }
    if (semver.satisfies(headlampVersion, pluginInfo.versionCompat)) {
      if (progressCallback) {
        progressCallback({ type: 'info', message: 'Headlamp version is compatible' });
      }
    } else {
      throw new Error('Headlamp version is not compatible with the plugin');
    }
  }

  if (signal && signal.aborted) {
    throw new Error('Download cancelled');
  }

  // Create temporary folder for extraction
  const tempDir = await fs.mkdtempSync(path.join(os.tmpdir(), 'headlamp-plugin-temp-'));
  // Defaulting to '' should never happen if recursive is true. So this is for the type
  // checker only.
  const tempFolder = fs.mkdirSync(path.join(tempDir, pluginName), { recursive: true }) ?? '';

  // First, download and extract the main archive
  if (progressCallback) {
    progressCallback({ type: 'info', message: 'Downloading main plugin archive' });
  }

  await downloadAndExtractSingleArchive(
    pluginInfo.archiveURL,
    pluginInfo.archiveChecksum,
    tempFolder,
    progressCallback,
    signal
  );

  await downloadExtraFiles(pluginInfo.extraFiles, tempFolder, progressCallback, signal);

  // Add artifacthub metadata to the plugin
  const packageJSON = JSON.parse(fs.readFileSync(`${tempFolder}/package.json`, 'utf8'));
  packageJSON.artifacthub = {
    name: pluginName,
    title: pluginInfo.display_name,
    url: `https://artifacthub.io/packages/headlamp/${pluginInfo.repository.name}/${pluginName}`,
    version: pluginInfo.version,
    repoName: pluginInfo.repository.name,
    author: pluginInfo.repository.user_alias,
  };
  packageJSON.isManagedByHeadlampPlugin = true;
  fs.writeFileSync(`${tempFolder}/package.json`, JSON.stringify(packageJSON, null, 2));

  return [pluginName, tempFolder];
}
```

---

</SwmSnippet>

<SwmSnippet path="/app/electron/plugin-management.ts" line="279">

---

We just returned from <SwmToken path="app/electron/plugin-management.ts" pos="272:14:14" line-data="      const [_, tempFolder] = await downloadExtractArchive(">`downloadExtractArchive`</SwmToken> in <SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>. Now, we make sure the destination folder exists, remove the old plugin folder, create a new one, and move the extracted plugin files in. This guarantees the plugin is fully replaced and avoids leftover files from previous versions.

```typescript
      // sleep(2000);  // comment out for testing

      // create the destination folder if it doesn't exist
      if (!fs.existsSync(destinationFolder)) {
        fs.mkdirSync(destinationFolder, { recursive: true });
      }

      // remove the existing plugin folder
      fs.rmdirSync(pluginDir, { recursive: true });

      // create the plugin folder
      fs.mkdirSync(pluginDir, { recursive: true });

      // move the plugin to the destination folder
      moveDirs(tempFolder, pluginDir);
      if (progressCallback) {
        progressCallback({ type: 'success', message: 'Plugin Updated' });
      }
    } catch (e) {
      if (progressCallback) {
        progressCallback({ type: 'error', message: e instanceof Error ? e.message : String(e) });
      } else {
        throw e;
      }
    }
  }
```

---

</SwmSnippet>

### Finalizing Multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Subscriptions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["WebSocket subscription fails"] --> node2{"Is retry count less than 3?"}
    click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:278:279"
    click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:279:283"
    node2 -->|"Yes"| node3["Log error and increment retry count, then retry subscription"]
    click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:281:282"
    node2 -->|"No"| node4["Do not retry subscription"]
    click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:283:284"
    node3 --> node5["Wait for next subscription event"]
    node4 --> node5
    node5 --> node6["Component unmounts or effect re-runs"]
    click node6 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:288:291"
    subgraph loop1["For each active subscription"]
      node6 --> node7["Clean up subscription"]
      click node7 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:290:291"
    end
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> subscription fails"] --> node2{"Is retry count less than 3?"}
%%     click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:278:279"
%%     click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:279:283"
%%     node2 -->|"Yes"| node3["Log error and increment retry count, then retry subscription"]
%%     click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:281:282"
%%     node2 -->|"No"| node4["Do not retry subscription"]
%%     click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:283:284"
%%     node3 --> node5["Wait for next subscription event"]
%%     node4 --> node5
%%     node5 --> node6["Component unmounts or effect <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="288:11:13" line-data="    // Cleanup subscriptions when effect re-runs or unmounts">`re-runs`</SwmToken>"]
%%     click node6 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:288:291"
%%     subgraph loop1["For each active subscription"]
%%       node6 --> node7["Clean up subscription"]
%%       click node7 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:290:291"
%%     end
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="277">

---

We just returned from <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken>. If multiplexing isn't enabled, <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="127:4:4" line-data="export function useWatchKubeObjectLists&lt;K extends KubeObject&gt;({">`useWatchKubeObjectLists`</SwmToken> falls back to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="150:3:3" line-data="    return useWatchKubeObjectListsLegacy({">`useWatchKubeObjectListsLegacy`</SwmToken>, which sets up individual <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="281:6:6" line-data="            console.error(&#39;WebSocket subscription failed:&#39;, error);">`WebSocket`</SwmToken> connections for each watched resource.

```typescript
          // Track retry count in the URL's searchParams
          const retryCount = parseInt(parsedUrl.searchParams.get('retryCount') || '0');
          if (retryCount < 3) {
            // Only log and allow retry if under threshold
            console.error('WebSocket subscription failed:', error);
            parsedUrl.searchParams.set('retryCount', (retryCount + 1).toString());
          }
        }
      );
    });

    // Cleanup subscriptions when effect re-runs or unmounts
    return () => {
      cleanups.forEach(cleanup => cleanup());
    };
  }, [connections, endpoint, handleUpdate]);
}
```

---

</SwmSnippet>

## Setting Up Legacy <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="150">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="150:3:3" line-data="    return useWatchKubeObjectListsLegacy({">`useWatchKubeObjectListsLegacy`</SwmToken> builds a connection object for each watched resource and sets up an <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="333:1:1" line-data="        onMessage(update: KubeListUpdateEvent&lt;K&gt;) {">`onMessage`</SwmToken> handler to update the cache. Then it calls <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="357:1:1" line-data="  useWebSockets&lt;KubeListUpdateEvent&lt;K&gt;&gt;({">`useWebSockets`</SwmToken> to actually open the connections and start listening for updates.

```typescript
    return useWatchKubeObjectListsLegacy({
      kubeObjectClass,
      endpoint,
      lists,
      queryParams,
    });
  }
}
```

---

</SwmSnippet>

# Managing Multiple <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Is API endpoint provided?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:321:322"
  node1 -->|"Yes"| node4["Aggregate connections for all lists"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:320:355"
  node1 -->|"No"| node5["No monitoring set up"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:321:322"

  subgraph loop1["For each entry in lists (cluster/namespace/resourceVersion)"]
    node4 --> node2["Set up WebSocket connection for resource list"]
    click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:323:353"
    node2 --> node3["Update resource list in real time on changes"]
    click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:334:351"
  end
  node4 --> node6["Enable real-time monitoring for all connections"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:357:360"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Is API endpoint provided?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:321:322"
%%   node1 -->|"Yes"| node4["Aggregate connections for all lists"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:320:355"
%%   node1 -->|"No"| node5["No monitoring set up"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:321:322"
%% 
%%   subgraph loop1["For each entry in lists (cluster/namespace/resourceVersion)"]
%%     node4 --> node2["Set up <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection for resource list"]
%%     click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:323:353"
%%     node2 --> node3["Update resource list in real time on changes"]
%%     click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:334:351"
%%   end
%%   node4 --> node6["Enable real-time monitoring for all connections"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:357:360"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="303">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="357:1:1" line-data="  useWebSockets&lt;KubeListUpdateEvent&lt;K&gt;&gt;({">`useWebSockets`</SwmToken>, we loop through connection requests, use a <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken> to avoid duplicate sockets, and set up listeners for each. If a connection isn't open, we call <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="503:6:6" line-data="export async function openWebSocket&lt;T&gt;(">`openWebSocket`</SwmToken> to start it.

```typescript
function useWatchKubeObjectListsLegacy<K extends KubeObject>({
  kubeObjectClass,
  endpoint,
  lists,
  queryParams,
}: {
  /** KubeObject class of the watched resource list */
  kubeObjectClass: (new (...args: any) => K) & typeof KubeObject<any>;
  /** Query parameters for the WebSocket connection URL */
  queryParams?: QueryParameters;
  /** Kube resource API endpoint information */
  endpoint?: KubeObjectEndpoint | null;
  /** Which clusters and namespaces to watch */
  lists: Array<{ cluster: string; namespace?: string; resourceVersion: string }>;
}) {
  const client = useQueryClient();

  const connections = useMemo(() => {
    if (!endpoint) return [];

    return lists.map(({ cluster, namespace, resourceVersion }) => {
      const url = makeUrl([KubeObjectEndpoint.toUrl(endpoint!, namespace)], {
        ...queryParams,
        watch: 1,
        resourceVersion,
      });

      return {
        cluster,
        url,
        onMessage(update: KubeListUpdateEvent<K>) {
          const key = kubeObjectListQuery<K>(
            kubeObjectClass,
            endpoint,
            namespace,
            cluster,
            queryParams ?? {}
          ).queryKey;
          client.setQueryData(key, (oldResponse: ListResponse<any> | undefined | null) => {
            if (!oldResponse) return oldResponse;

            const newList = KubeList.applyUpdate(
              oldResponse.list,
              update,
              kubeObjectClass,
              cluster
            );
            return { ...oldResponse, list: newList };
          });
        },
      };
    });
  }, [lists, kubeObjectClass, endpoint]);

  useWebSockets<KubeListUpdateEvent<K>>({
    enabled: !!endpoint,
    connections,
  });
}
```

---

</SwmSnippet>

# Opening Cluster-Aware <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Are WebSocket connections enabled?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:585:587"
  node1 -->|"Yes"| node2["Map over connections to open/manage each WebSocket"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:642:643"
  node1 -->|"No"| node5["Skip connection setup"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:586:587"
  
  subgraph loop1["For each connection in connections"]
    node2 --> node4["Cleaning Up Single WebSocket Subscriptions"]
    
    node4 -->|"No listeners remain"| node3["Connection Key Management and Listener Reference Counting"]
    
  end
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node4 goToHeading "Processing Incoming WebSocket Messages"
node4:::HeadingStyle
click node4 goToHeading "Managing Single WebSocket Connection Lifecycle"
node4:::HeadingStyle
click node4 goToHeading "Cleaning Up Single WebSocket Subscriptions"
node4:::HeadingStyle
click node3 goToHeading "Connection Key Management and Listener Reference Counting"
node3:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Are <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections enabled?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:585:587"
%%   node1 -->|"Yes"| node2["Map over connections to open/manage each <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:642:643"
%%   node1 -->|"No"| node5["Skip connection setup"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:586:587"
%%   
%%   subgraph loop1["For each connection in connections"]
%%     node2 --> node4["Cleaning Up Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Subscriptions"]
%%     
%%     node4 -->|"No listeners remain"| node3["Connection Key Management and Listener Reference Counting"]
%%     
%%   end
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node4 goToHeading "Processing Incoming <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Messages"
%% node4:::HeadingStyle
%% click node4 goToHeading "Managing Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Lifecycle"
%% node4:::HeadingStyle
%% click node4 goToHeading "Cleaning Up Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Subscriptions"
%% node4:::HeadingStyle
%% click node3 goToHeading "Connection Key Management and Listener Reference Counting"
%% node3:::HeadingStyle
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="566">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="602:1:1" line-data="        openWebSocket(url, { protocols, type, cluster, onMessage })">`openWebSocket`</SwmToken> builds the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="576:15:15" line-data="   * Any additional protocols to include in WebSocket connection">`WebSocket`</SwmToken> URL, adds cluster context if needed, fetches kubeconfig for authorization, and sets up protocol strings. Incoming messages are parsed as JSON or binary based on the type, then handed off to the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="591:13:13" line-data="    function connect({ cluster, url, onMessage }: WebSocketConnectionRequest&lt;T&gt;) {">`onMessage`</SwmToken> callback.

```typescript
export function useWebSockets<T>({
  connections,
  enabled = true,
  protocols,
  type = 'json',
}: {
  enabled?: boolean;
  /** Make sure that connections value is stable between renders */
  connections: Array<WebSocketConnectionRequest<T>>;
  /**
   * Any additional protocols to include in WebSocket connection
   * make sure that the value is stable between renders
   */
  protocols?: string | string[];
  /**
   * Type of websocket data
   */
  type?: 'json' | 'binary';
}) {
  useEffect(() => {
    if (!enabled) return;

    let isCurrent = true;

    /** Open a connection to websocket */
    function connect({ cluster, url, onMessage }: WebSocketConnectionRequest<T>) {
      const connectionKey = cluster + url;

      if (!sockets.has(connectionKey)) {
        // Add new listener for this URL
        listeners.set(connectionKey, [...(listeners.get(connectionKey) ?? []), onMessage]);

        // Mark socket as pending, so we don't open more than one
        sockets.set(connectionKey, 'pending');

        let ws: WebSocket | undefined;
        openWebSocket(url, { protocols, type, cluster, onMessage })
          .then(socket => {
```

---

</SwmSnippet>

## Processing Incoming <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Messages

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start: Prepare connection details"] --> node2{"Is cluster specified?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:529:532"
  node2 -->|"Yes"| node3["Find kubeconfig for cluster"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:532:535"
  node2 -->|"No"| node5["Build protocols and open WebSocket"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:535:538"
  node3 --> node4{"Is kubeconfig found?"}
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:538:541"
  node4 -->|"Yes"| node6["Add user authorization protocol"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:539:541"
  node4 -->|"No kubeconfig found"| node5["Build protocols and open WebSocket"]
  node6 --> node5
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:547:552"
  node5 --> node7{"Message type: JSON or binary?"}
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node7 -->|"JSON"| node8["Process message as JSON"]
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node7 -->|"Binary"| node9["Process message as binary"]
  click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node8 --> node10["Handle incoming message"]
  click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:551:552"
  node9 --> node10

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start: Prepare connection details"] --> node2{"Is cluster specified?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:529:532"
%%   node2 -->|"Yes"| node3["Find kubeconfig for cluster"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:532:535"
%%   node2 -->|"No"| node5["Build protocols and open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:535:538"
%%   node3 --> node4{"Is kubeconfig found?"}
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:538:541"
%%   node4 -->|"Yes"| node6["Add user authorization protocol"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:539:541"
%%   node4 -->|"No kubeconfig found"| node5["Build protocols and open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>"]
%%   node6 --> node5
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:547:552"
%%   node5 --> node7{"Message type: JSON or binary?"}
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node7 -->|"JSON"| node8["Process message as JSON"]
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node7 -->|"Binary"| node9["Process message as binary"]
%%   click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node8 --> node10["Handle incoming message"]
%%   click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:551:552"
%%   node9 --> node10
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="503">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="509:1:1" line-data="    onMessage,">`onMessage`</SwmToken> in <SwmPath>[frontend/…/v2/hooks.ts](frontend/src/lib/k8s/api/v2/hooks.ts)</SwmPath> updates the client cache with new object data for non-ADDED events. This keeps the UI in sync with server changes as messages arrive from the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="512:15:15" line-data="     * Any additional protocols to include in WebSocket connection">`WebSocket`</SwmToken>.

```typescript
export async function openWebSocket<T>(
  url: string,
  {
    protocols: moreProtocols = [],
    type = 'binary',
    cluster = getCluster() ?? '',
    onMessage,
  }: {
    /**
     * Any additional protocols to include in WebSocket connection
     */
    protocols?: string | string[];
    /**
     *
     */
    type: 'json' | 'binary';
    /**
     * Cluster name
     */
    cluster?: string;
    /**
     * Message callback
     */
    onMessage: (data: T) => void;
  }
) {
  const path = [url];
  const protocols = ['base64.binary.k8s.io', ...(moreProtocols ?? [])];

  if (cluster) {
    path.unshift('clusters', cluster);

    try {
      const kubeconfig = await findKubeconfigByClusterName(cluster);

      if (kubeconfig !== null) {
        const userID = getUserIdFromLocalStorage();
        protocols.push(`base64url.headlamp.authorization.k8s.io.${userID}`);
      }
    } catch (error) {
      console.error('Error while finding kubeconfig:', error);
    }
  }

  const socket = new WebSocket(makeUrl([getBaseWsUrl(), ...path], {}), protocols);
  socket.binaryType = 'arraybuffer';
  socket.addEventListener('message', (body: MessageEvent) => {
    const data = type === 'json' ? JSON.parse(body.data) : body.data;
    onMessage(data);
  });
  socket.addEventListener('error', error => {
    console.error('WebSocket error:', error);
  });

  return socket;
}
```

---

</SwmSnippet>

## Managing Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Lifecycle

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/hooks.ts" line="152">

---

We parse messages and handle errors before passing data to the cache updater.

```typescript
    onMessage(update: KubeListUpdateEvent<K>) {
      if (update.type !== 'ADDED' && update.object) {
        client.setQueryData(queryKey, new kubeObjectClass(update.object));
      }
    },
```

---

</SwmSnippet>

## Cleaning Up Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Subscriptions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Are real-time updates enabled and URL valid?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:458:460"
  node1 -->|"Yes"| node2["Connect to WebSocket for live updates"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:464:479"
  node1 -->|"No"| node6["Skip real-time updates"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:459:460"
  subgraph connectionLifecycle["While connection is active"]
    node2 --> node3["Deliver incoming updates to business logic (onMessage)"]
    click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:437:449"
    node2 --> node4["Handle connection or message errors (onError)"]
    click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:450:452"
  end
  node2 --> node5["Clean up connection when updates are no longer needed"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:481:485"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Are real-time updates enabled and URL valid?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:458:460"
%%   node1 -->|"Yes"| node2["Connect to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> for live updates"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:464:479"
%%   node1 -->|"No"| node6["Skip real-time updates"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:459:460"
%%   subgraph connectionLifecycle["While connection is active"]
%%     node2 --> node3["Deliver incoming updates to business logic (<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="333:1:1" line-data="        onMessage(update: KubeListUpdateEvent&lt;K&gt;) {">`onMessage`</SwmToken>)"]
%%     click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:437:449"
%%     node2 --> node4["Handle connection or message errors (<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="421:1:1" line-data="  onError,">`onError`</SwmToken>)"]
%%     click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:450:452"
%%   end
%%   node2 --> node5["Clean up connection when updates are no longer needed"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:481:485"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="416">

---

We just returned from the async <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="423:11:11" line-data="  /** Function that returns the WebSocket URL to connect to */">`WebSocket`</SwmToken> connection logic in <SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>. Now, we clean up listeners and sockets if the hook is unmounted or if no listeners remain, making sure we don't leak connections.

```typescript
export function useWebSocket<T>({
  url: createUrl,
  enabled = true,
  cluster = '',
  onMessage,
  onError,
}: {
  /** Function that returns the WebSocket URL to connect to */
  url: () => string;
  /** Whether the WebSocket connection should be active */
  enabled?: boolean;
  /** The Kubernetes cluster ID to watch */
  cluster?: string;
  /** Callback function to handle incoming messages */
  onMessage: (data: T) => void;
  /** Callback function to handle connection errors */
  onError?: (error: Error) => void;
}) {
  const url = useMemo(() => (enabled ? createUrl() : ''), [enabled, createUrl]);

  const stableOnMessage = useCallback(
    (rawData: any) => {
      try {
        let parsedData: T;
        try {
          parsedData = typeof rawData === 'string' ? JSON.parse(rawData) : rawData;
        } catch (parseError) {
          console.error('Failed to parse WebSocket message:', parseError);
          onError?.(parseError as Error);
          return;
        }

        onMessage(parsedData);
      } catch (err) {
        console.error('Failed to process WebSocket message:', err);
        onError?.(err as Error);
      }
    },
    [onMessage, onError]
  );

```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="457">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="423:17:17" line-data="  /** Function that returns the WebSocket URL to connect to */">`connect`</SwmToken>, we use a <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken> to manage sockets and listeners. We add listeners for each connection, mark sockets as 'pending' while opening, and only open a new socket if one isn't already tracked for that key.

```typescript
  useEffect(() => {
    if (!enabled || !url) {
      return;
    }

    let cleanup: (() => void) | undefined;

    const connectWebSocket = async () => {
      try {
        const parsedUrl = new URL(url, getBaseWsUrl());
        cleanup = await WebSocketManager.subscribe(
          cluster,
          parsedUrl.pathname,
          parsedUrl.search.slice(1),
          stableOnMessage
        );
      } catch (err) {
        console.error('WebSocket connection failed:', err);
        onError?.(err as Error);
      }
    };

    connectWebSocket();

    return () => {
      if (cleanup) {
        cleanup();
      }
    };
  }, [url, enabled, cluster, stableOnMessage, onError]);
}
```

---

</SwmSnippet>

## Connection Key Management and Listener Reference Counting

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start: Manage WebSocket connections for endpoints"]
    click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:604:649"
    
    subgraph loop1["For each endpoint in connections"]
      node1 --> node2["Connect and set up listener"]
      click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:604:615"
      node2 --> node3{"Was hook unmounted during connection?"}
      click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:608:612"
      node3 -->|"Yes"| node4["Close WebSocket and clean up"]
      click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:609:611"
      node3 -->|"No"| node5["Store active connection"]
      click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:614:614"
      node5 --> node6["On cleanup, remove listener"]
      click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:621:627"
      node6 --> node7{"Are listeners left for this connection?"}
      click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:630:638"
      node7 -->|"No"| node8["Close WebSocket and clean up"]
      click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:631:637"
    end
    
    node1 --> node9["On hook unmount, run all disconnect callbacks"]
    click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:644:647"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Start: Manage <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections for endpoints"]
%%     click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:604:649"
%%     
%%     subgraph loop1["For each endpoint in connections"]
%%       node1 --> node2["Connect and set up listener"]
%%       click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:604:615"
%%       node2 --> node3{"Was hook unmounted during connection?"}
%%       click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:608:612"
%%       node3 -->|"Yes"| node4["Close <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> and clean up"]
%%       click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:609:611"
%%       node3 -->|"No"| node5["Store active connection"]
%%       click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:614:614"
%%       node5 --> node6["On cleanup, remove listener"]
%%       click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:621:627"
%%       node6 --> node7{"Are listeners left for this connection?"}
%%       click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:630:638"
%%       node7 -->|"No"| node8["Close <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> and clean up"]
%%       click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:631:637"
%%     end
%%     
%%     node1 --> node9["On hook unmount, run all disconnect callbacks"]
%%     click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:644:647"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="604">

---

We just returned from the connection setup in <SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>. Now, we use the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="608:5:5" line-data="            if (!isCurrent) {">`isCurrent`</SwmToken> flag to check if the hook is still mounted when the socket opens. If not, we close and clean up the socket. We also remove listeners and close sockets when no listeners remain, keeping resource usage tight.

```typescript
            ws = socket;

            // Hook was unmounted while it was connecting to WebSocket
            // so we close the socket and clean up
            if (!isCurrent) {
              ws.close();
              sockets.delete(connectionKey);
              return;
            }

            sockets.set(connectionKey, ws);
          })
          .catch(err => {
            console.error(err);
          });
      }

      return () => {
        const connectionKey = cluster + url;

        // Clean up the listener
        const newListeners = listeners.get(connectionKey)?.filter(it => it !== onMessage) ?? [];
        listeners.set(connectionKey, newListeners);

        // No one is listening to the connection
        // so we can close it
        if (newListeners.length === 0) {
          const maybeExisting = sockets.get(connectionKey);
          if (maybeExisting) {
            if (maybeExisting !== 'pending') {
              maybeExisting.close();
            }
            sockets.delete(connectionKey);
          }
        }
      };
    }

    const disconnectCallbacks = connections.map(endpoint => connect(endpoint));

    return () => {
      isCurrent = false;
      disconnectCallbacks.forEach(fn => fn());
    };
  }, [enabled, type, connections, protocols]);
}
```

---

</SwmSnippet>

# Connection Lifecycle and Cleanup

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Request WebSocket connection for cluster+URL"]
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:591:594"
  node1 --> node2{"Is there already a connection?"}
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:594:594"
  node2 -->|"No"| node3["Add listener and mark connection as pending"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:595:599"
  node3 --> node4["Open WebSocket connection"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:602:615"
  node4 --> node5{"Is connection still needed when open?"}
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:608:613"
  node5 -->|"No"| node6["Close WebSocket and clean up"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:609:611"
  node5 -->|"Yes"| node7["Store active WebSocket"]
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:614:614"
  node2 -->|"Yes"| node7
  node7 --> node8["Return cleanup function"]
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:621:639"
  node6 --> node8
  node8 --> node9{"Are there any listeners left?"}
  click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:630:638"
  node9 -->|"No"| node10["Close connection and remove from registry"]
  click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:631:637"
  node9 -->|"Yes"| node11["Keep connection open"]
  click node11 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:625:627"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Request <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection for cluster+URL"]
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:591:594"
%%   node1 --> node2{"Is there already a connection?"}
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:594:594"
%%   node2 -->|"No"| node3["Add listener and mark connection as pending"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:595:599"
%%   node3 --> node4["Open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:602:615"
%%   node4 --> node5{"Is connection still needed when open?"}
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:608:613"
%%   node5 -->|"No"| node6["Close <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> and clean up"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:609:611"
%%   node5 -->|"Yes"| node7["Store active <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>"]
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:614:614"
%%   node2 -->|"Yes"| node7
%%   node7 --> node8["Return cleanup function"]
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:621:639"
%%   node6 --> node8
%%   node8 --> node9{"Are there any listeners left?"}
%%   click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:630:638"
%%   node9 -->|"No"| node10["Close connection and remove from registry"]
%%   click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:631:637"
%%   node9 -->|"Yes"| node11["Keep connection open"]
%%   click node11 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:625:627"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="591">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="591:3:3" line-data="    function connect({ cluster, url, onMessage }: WebSocketConnectionRequest&lt;T&gt;) {">`connect`</SwmToken>, we use a <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken> to manage sockets and listeners. We add listeners for each connection, mark sockets as 'pending' while opening, and only open a new socket if one isn't already tracked for that key.

```typescript
    function connect({ cluster, url, onMessage }: WebSocketConnectionRequest<T>) {
      const connectionKey = cluster + url;

      if (!sockets.has(connectionKey)) {
        // Add new listener for this URL
        listeners.set(connectionKey, [...(listeners.get(connectionKey) ?? []), onMessage]);

        // Mark socket as pending, so we don't open more than one
        sockets.set(connectionKey, 'pending');

        let ws: WebSocket | undefined;
        openWebSocket(url, { protocols, type, cluster, onMessage })
          .then(socket => {
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="604">

---

We just returned from the connection setup in <SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>. Now, we use the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="608:5:5" line-data="            if (!isCurrent) {">`isCurrent`</SwmToken> flag to check if the hook is still mounted when the socket opens. If not, we close and clean up the socket. We also remove listeners and close sockets when no listeners remain, keeping resource usage tight.

```typescript
            ws = socket;

            // Hook was unmounted while it was connecting to WebSocket
            // so we close the socket and clean up
            if (!isCurrent) {
              ws.close();
              sockets.delete(connectionKey);
              return;
            }

            sockets.set(connectionKey, ws);
          })
          .catch(err => {
            console.error(err);
          });
      }

      return () => {
        const connectionKey = cluster + url;

        // Clean up the listener
        const newListeners = listeners.get(connectionKey)?.filter(it => it !== onMessage) ?? [];
        listeners.set(connectionKey, newListeners);

        // No one is listening to the connection
        // so we can close it
        if (newListeners.length === 0) {
          const maybeExisting = sockets.get(connectionKey);
          if (maybeExisting) {
            if (maybeExisting !== 'pending') {
              maybeExisting.close();
            }
            sockets.delete(connectionKey);
          }
        }
      };
    }
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBdHlwZXNjcmlwdC1oZWFkbGFtcCUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="typescript-headlamp"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
