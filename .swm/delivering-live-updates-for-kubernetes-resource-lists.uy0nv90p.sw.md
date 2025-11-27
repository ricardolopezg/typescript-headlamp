---
title: Delivering Live Updates for Kubernetes Resource Lists
---
This document describes how the system keeps Kubernetes resource lists up to date by delivering live updates. The flow chooses between an optimized multiplexed strategy and a legacy approach based on system capabilities. When multiplexing is enabled, multiple resource lists are watched over a single connection, and plugin updates are managed as needed. If multiplexing is not available, the system sets up individual connections for each resource list.

# Choosing the Watch Strategy

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="127">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="127:4:4" line-data="export function useWatchKubeObjectLists&lt;K extends KubeObject&gt;({">`useWatchKubeObjectLists`</SwmToken>, we start by checking if <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> multiplexing is enabled. If it is, we delegate to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, which handles multiple resource watches over a single connection. This sets up the rest of the flow to use the optimized path for resource watching.

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

## Setting Up Multiplexed Resource Watches

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start: Prepare to watch multiple Kubernetes object lists"]
  click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:170:180"
  node1 --> node2{"Are there valid endpoints and connections?"}
  click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:261:263"
  node2 -->|"Yes"| loop1
  node2 -->|"No"| node4["No subscriptions established"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:262:263"
  
  subgraph loop1["For each connection"]
    node5["Subscribe and listen for updates"]
    click node5 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:268:276"
    node5 --> node6["Handling Plugin Update Events"]
    
    node6 --> node7{"Did subscription fail?"}
    click node7 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:277:283"
    node7 -->|"Retry count < 3"| node8["Retry subscription"]
    click node8 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:281:283"
    node7 -->|"Retry count >= 3"| node9["Stop retrying"]
    click node9 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:284:285"
  end
  loop1 --> node10["Cleanup subscriptions when done"]
  click node10 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:289:292"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node6 goToHeading "Handling Plugin Update Events"
node6:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start: Prepare to watch multiple Kubernetes object lists"]
%%   click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:170:180"
%%   node1 --> node2{"Are there valid endpoints and connections?"}
%%   click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:261:263"
%%   node2 -->|"Yes"| loop1
%%   node2 -->|"No"| node4["No subscriptions established"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:262:263"
%%   
%%   subgraph loop1["For each connection"]
%%     node5["Subscribe and listen for updates"]
%%     click node5 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:268:276"
%%     node5 --> node6["Handling Plugin Update Events"]
%%     
%%     node6 --> node7{"Did subscription fail?"}
%%     click node7 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:277:283"
%%     node7 -->|"Retry count < 3"| node8["Retry subscription"]
%%     click node8 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:281:283"
%%     node7 -->|"Retry count >= 3"| node9["Stop retrying"]
%%     click node9 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:284:285"
%%   end
%%   loop1 --> node10["Cleanup subscriptions when done"]
%%   click node10 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:289:292"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node6 goToHeading "Handling Plugin Update Events"
%% node6:::HeadingStyle
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="170">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="170:2:2" line-data="function useWatchKubeObjectListsMultiplexed&lt;K extends KubeObject&gt;({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, we prep all the connection info and handlers for each resource list, then set up <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="203:5:5" line-data="      // Construct WebSocket URL with current parameters">`WebSocket`</SwmToken> subscriptions. The next step involves calling into the Electron main process to handle updates that require backend coordination, like plugin management or privileged actions.

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

### Handling Plugin Update Events

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start plugin update process"]
  click node1 openCode "app/electron/main.ts:365:366"
  node1 --> node2{"Is plugin name provided?"}
  click node2 openCode "app/electron/main.ts:367:367"
  node2 -->|"No"| node3["Update cache with error: 'Plugin Name is required'"]
  click node3 openCode "app/electron/main.ts:368:372"
  node2 -->|"Yes"| node4["Update cache: status 'updating plugin'"]
  click node4 openCode "app/electron/main.ts:375:381"
  node4 --> node5["Initiate plugin update and report progress"]
  click node5 openCode "app/electron/main.ts:383:391"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start plugin update process"]
%%   click node1 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:365:366"
%%   node1 --> node2{"Is plugin name provided?"}
%%   click node2 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:367:367"
%%   node2 -->|"No"| node3["Update cache with error: 'Plugin Name is required'"]
%%   click node3 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:368:372"
%%   node2 -->|"Yes"| node4["Update cache: status 'updating plugin'"]
%%   click node4 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:375:381"
%%   node4 --> node5["Initiate plugin update and report progress"]
%%   click node5 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:383:391"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/app/electron/main.ts" line="365">

---

<SwmToken path="app/electron/main.ts" pos="365:3:3" line-data="  private handleUpdate(eventData: Action, updateCache: (progress: ProgressResp) =&gt; void) {">`handleUpdate`</SwmToken> in the Electron main process receives update events, sets up progress tracking, and then calls <SwmToken path="app/electron/main.ts" pos="383:1:3" line-data="    PluginManager.update(">`PluginManager.update`</SwmToken> to actually perform the plugin update. This handoff is needed because the update logic (downloading, extracting, replacing files) is centralized in <SwmToken path="app/electron/main.ts" pos="383:1:1" line-data="    PluginManager.update(">`PluginManager`</SwmToken> for consistency and error handling.

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

### Updating and Verifying Plugins

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start plugin update for pluginName"] --> node2["Get installed plugins in destinationFolder"]
  click node1 openCode "app/electron/plugin-management.ts:239:246"
  click node2 openCode "app/electron/plugin-management.ts:248:251"
  node2 --> node3{"Is plugin installed?"}
  click node3 openCode "app/electron/plugin-management.ts:252:255"
  node3 -->|"No"| node4["Error: Plugin not found"]
  click node4 openCode "app/electron/plugin-management.ts:254:255"
  node3 -->|"Yes"| node5["Fetch latest plugin info and check compatibility"]
  click node5 openCode "app/electron/plugin-management.ts:262:277"
  node5 --> node6{"Is newer version available? (Current: currentVersion, Latest: latestVersion)"}
  click node6 openCode "app/electron/plugin-management.ts:267:269"
  node6 -->|"No"| node7["Error: No updates available"]
  click node7 openCode "app/electron/plugin-management.ts:268:269"
  node6 -->|"Yes"| node8["Download, extract, and replace plugin files"]
  click node8 openCode "app/electron/plugin-management.ts:287:293"
  node8 --> node9["Report success"]
  click node9 openCode "app/electron/plugin-management.ts:294:296"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start plugin update for <SwmToken path="app/electron/main.ts" pos="366:8:8" line-data="    const { identifier, pluginName, destinationFolder, headlampVersion } = eventData;">`pluginName`</SwmToken>"] --> node2["Get installed plugins in <SwmToken path="app/electron/main.ts" pos="366:11:11" line-data="    const { identifier, pluginName, destinationFolder, headlampVersion } = eventData;">`destinationFolder`</SwmToken>"]
%%   click node1 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:239:246"
%%   click node2 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:248:251"
%%   node2 --> node3{"Is plugin installed?"}
%%   click node3 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:252:255"
%%   node3 -->|"No"| node4["Error: Plugin not found"]
%%   click node4 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:254:255"
%%   node3 -->|"Yes"| node5["Fetch latest plugin info and check compatibility"]
%%   click node5 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:262:277"
%%   node5 --> node6{"Is newer version available? (Current: <SwmToken path="app/electron/plugin-management.ts" pos="265:3:3" line-data="      const currentVersion = packageJson.artifacthub.version;">`currentVersion`</SwmToken>, Latest: <SwmToken path="app/electron/plugin-management.ts" pos="264:3:3" line-data="      const latestVersion = pluginData.version;">`latestVersion`</SwmToken>)"}
%%   click node6 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:267:269"
%%   node6 -->|"No"| node7["Error: No updates available"]
%%   click node7 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:268:269"
%%   node6 -->|"Yes"| node8["Download, extract, and replace plugin files"]
%%   click node8 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:287:293"
%%   node8 --> node9["Report success"]
%%   click node9 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:294:296"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/app/electron/plugin-management.ts" line="239">

---

In <SwmToken path="app/electron/main.ts" pos="383:1:3" line-data="    PluginManager.update(">`PluginManager.update`</SwmToken>, we look up the installed plugin, check its current version, fetch the latest info, and compare versions. If an update is needed, we download and extract the new version to a temp folder before replacing the old one. This relies on specific metadata and structure in the plugin directories and <SwmPath>[package.json](package.json)</SwmPath>.

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

<SwmToken path="app/electron/plugin-management.ts" pos="469:4:4" line-data="async function downloadExtractArchive(">`downloadExtractArchive`</SwmToken> handles compatibility, downloads, and adds Headlamp-specific metadata to the plugin's <SwmPath>[package.json](package.json)</SwmPath>.

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

Back in <SwmToken path="app/electron/main.ts" pos="383:1:3" line-data="    PluginManager.update(">`PluginManager.update`</SwmToken>, after extracting the new plugin, we remove the old plugin folder, recreate it, and move the new files in. This guarantees a clean update and avoids leftover files from previous versions.

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

### Finalizing Multiplexed Subscriptions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["WebSocket subscription fails"] --> node2{"Is retryCount less than 3?"}
    click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:278:279"
    node2 -->|"Yes"| node3["Log error and increment retryCount"]
    click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:279:283"
    node2 -->|"No"| node4["Do not retry"]
    click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:281:283"
    click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:284:285"
    subgraph loop1["For each active subscription"]
        node5["Cleanup subscription when effect re-runs or unmounts"]
        click node5 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:289:291"
    end
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> subscription fails"] --> node2{"Is <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="278:3:3" line-data="          const retryCount = parseInt(parsedUrl.searchParams.get(&#39;retryCount&#39;) || &#39;0&#39;);">`retryCount`</SwmToken> less than 3?"}
%%     click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:278:279"
%%     node2 -->|"Yes"| node3["Log error and increment <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="278:3:3" line-data="          const retryCount = parseInt(parsedUrl.searchParams.get(&#39;retryCount&#39;) || &#39;0&#39;);">`retryCount`</SwmToken>"]
%%     click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:279:283"
%%     node2 -->|"No"| node4["Do not retry"]
%%     click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:281:283"
%%     click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:284:285"
%%     subgraph loop1["For each active subscription"]
%%         node5["Cleanup subscription when effect <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="288:11:13" line-data="    // Cleanup subscriptions when effect re-runs or unmounts">`re-runs`</SwmToken> or unmounts"]
%%         click node5 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:289:291"
%%     end
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="277">

---

Back in <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, after handling updates via the Electron main process, we handle <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="281:6:6" line-data="            console.error(&#39;WebSocket subscription failed:&#39;, error);">`WebSocket`</SwmToken> subscription failures by retrying up to three times and clean up all subscriptions when the effect <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="288:11:13" line-data="    // Cleanup subscriptions when effect re-runs or unmounts">`re-runs`</SwmToken> or unmounts.

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

## Fallback to Legacy Watch Logic

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="150">

---

Back in <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="127:4:4" line-data="export function useWatchKubeObjectLists&lt;K extends KubeObject&gt;({">`useWatchKubeObjectLists`</SwmToken>, if multiplexing isn't enabled, we switch to the legacy watcher by calling <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="150:3:3" line-data="    return useWatchKubeObjectListsLegacy({">`useWatchKubeObjectListsLegacy`</SwmToken>. This keeps things working for environments that don't support multiplexing.

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

# Legacy Resource Watch Setup

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start watching Kubernetes resource lists"] --> node2{"Is endpoint provided?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:303:318"
  click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:320:321"
  node2 -->|"No"| node6["No live updates"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:321:321"
  node2 -->|"Yes"| node3["Set up live connections"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:323:355"

  subgraph loop1["For each cluster/namespace/resourceVersion in lists"]
    node3 --> node4["Create connection and listen for updates"]
    click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:323:353"
    node4 --> node5{"Is there an existing resource list?"}
    click node5 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:341:342"
    node5 -->|"Yes"| node8["Apply update to resource list"]
    click node8 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:344:350"
    node8 --> node7["Resource lists stay up to date"]
    click node7 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:351:351"
    node5 -->|"No"| node3
  end

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start watching Kubernetes resource lists"] --> node2{"Is endpoint provided?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:303:318"
%%   click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:320:321"
%%   node2 -->|"No"| node6["No live updates"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:321:321"
%%   node2 -->|"Yes"| node3["Set up live connections"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:323:355"
%% 
%%   subgraph loop1["For each cluster/namespace/resourceVersion in lists"]
%%     node3 --> node4["Create connection and listen for updates"]
%%     click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:323:353"
%%     node4 --> node5{"Is there an existing resource list?"}
%%     click node5 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:341:342"
%%     node5 -->|"Yes"| node8["Apply update to resource list"]
%%     click node8 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:344:350"
%%     node8 --> node7["Resource lists stay up to date"]
%%     click node7 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:351:351"
%%     node5 -->|"No"| node3
%%   end
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="303">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="303:2:2" line-data="function useWatchKubeObjectListsLegacy&lt;K extends KubeObject&gt;({">`useWatchKubeObjectListsLegacy`</SwmToken> maps each resource list to a <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="311:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection config and delegates the connection and update handling to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="357:1:1" line-data="  useWebSockets&lt;KubeListUpdateEvent&lt;K&gt;&gt;({">`useWebSockets`</SwmToken>. This keeps the legacy logic simple and focused on setup.

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

# Managing <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Are WebSocket connections enabled?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:585:586"
  node1 -->|"Yes"| node2["Manage WebSocket connections"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:642:643"
  node1 -->|"No"| node5["Skip connection setup"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:586:587"

  subgraph loop1["For each connection in connections"]
    node2 --> node3{"Is connection already open or pending?"}
    
    node3 -->|"No"| node4["Opening and Authorizing WebSocket Connections"]
    
    node3 -->|"Yes"| node6["Cleaning Up Shared WebSocket Connections"]
    
    node4 --> node7["Cleaning Up Shared WebSocket Connections"]
    
  end

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node3 goToHeading "Cleaning Up Shared WebSocket Connections"
node3:::HeadingStyle
click node4 goToHeading "Opening and Authorizing WebSocket Connections"
node4:::HeadingStyle
click node6 goToHeading "Cleaning Up Shared WebSocket Connections"
node6:::HeadingStyle
click node7 goToHeading "Cleaning Up Shared WebSocket Connections"
node7:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Are <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections enabled?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:585:586"
%%   node1 -->|"Yes"| node2["Manage <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:642:643"
%%   node1 -->|"No"| node5["Skip connection setup"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:586:587"
%% 
%%   subgraph loop1["For each connection in connections"]
%%     node2 --> node3{"Is connection already open or pending?"}
%%     
%%     node3 -->|"No"| node4["Opening and Authorizing <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"]
%%     
%%     node3 -->|"Yes"| node6["Cleaning Up Shared <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"]
%%     
%%     node4 --> node7["Cleaning Up Shared <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"]
%%     
%%   end
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node3 goToHeading "Cleaning Up Shared <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%% node3:::HeadingStyle
%% click node4 goToHeading "Opening and Authorizing <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%% node4:::HeadingStyle
%% click node6 goToHeading "Cleaning Up Shared <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%% node6:::HeadingStyle
%% click node7 goToHeading "Cleaning Up Shared <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections"
%% node7:::HeadingStyle
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="566">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="566:4:4" line-data="export function useWebSockets&lt;T&gt;({">`useWebSockets`</SwmToken>, we manage <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="576:15:15" line-data="   * Any additional protocols to include in WebSocket connection">`WebSocket`</SwmToken> connections and listeners using global maps keyed by cluster and URL. This lets us share connections and clean up when no listeners remain. The next step is to actually open the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="576:15:15" line-data="   * Any additional protocols to include in WebSocket connection">`WebSocket`</SwmToken> and handle messages.

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

## Opening and Authorizing <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Prepare WebSocket connection parameters"]
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:529:532"
  node1 --> node2{"Is a cluster specified?"}
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:532:545"
  node2 -->|"Yes"| node3["Add cluster info to connection"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:532:545"
  node3 --> node4{"Is kubeconfig found for cluster?"}
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:535:542"
  node4 -->|"Yes"| node9["Add user authorization to connection"]
  click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:539:541"
  node9 --> node5["Establish WebSocket connection"]
  node4 -->|"No"| node5["Establish WebSocket connection"]
  node2 -->|"No"| node5["Establish WebSocket connection"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:547:557"
  node5 --> node6{"Is message type 'json'?"}
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node6 -->|"Yes"| node7["Parse incoming messages as JSON and deliver to app"]
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node6 -->|"No"| node8["Deliver incoming messages as binary to app"]
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node7 --> node10["Return WebSocket connection"]
  node8 --> node10
  click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:557:558"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Prepare <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection parameters"]
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:529:532"
%%   node1 --> node2{"Is a cluster specified?"}
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:532:545"
%%   node2 -->|"Yes"| node3["Add cluster info to connection"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:532:545"
%%   node3 --> node4{"Is kubeconfig found for cluster?"}
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:535:542"
%%   node4 -->|"Yes"| node9["Add user authorization to connection"]
%%   click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:539:541"
%%   node9 --> node5["Establish <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   node4 -->|"No"| node5["Establish <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   node2 -->|"No"| node5["Establish <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:547:557"
%%   node5 --> node6{"Is message type 'json'?"}
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node6 -->|"Yes"| node7["Parse incoming messages as JSON and deliver to app"]
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node6 -->|"No"| node8["Deliver incoming messages as binary to app"]
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node7 --> node10["Return <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   node8 --> node10
%%   click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:557:558"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="503">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="503:6:6" line-data="export async function openWebSocket&lt;T&gt;(">`openWebSocket`</SwmToken> builds the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="512:15:15" line-data="     * Any additional protocols to include in WebSocket connection">`WebSocket`</SwmToken> URL and protocol list based on cluster and user info, then opens the connection and sets up message handling. The next step is to process incoming messages, which happens in the hooks layer.

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

## Processing Incoming Resource Updates

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/hooks.ts" line="152">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="333:1:1" line-data="        onMessage(update: KubeListUpdateEvent&lt;K&gt;) {">`onMessage`</SwmToken> in the hooks layer processes incoming updates, updating the React Query cache with new resource data. After this, control returns to the <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> logic for connection management.

```typescript
    onMessage(update: KubeListUpdateEvent<K>) {
      if (update.type !== 'ADDED' && update.object) {
        client.setQueryData(queryKey, new kubeObjectClass(update.object));
      }
    },
```

---

</SwmSnippet>

## Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Management

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Is WebSocket connection enabled and URL valid?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:434:460"
  node1 -->|"Yes"| node2["Connect to Kubernetes cluster and listen for messages"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:464:473"
  node1 -->|"No"| node5["Do not connect"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:458:460"
  node2 --> node3["Process each incoming message and handle errors"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:437:452"
  node2 -->|"On cleanup"| node4["Disconnect from cluster"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:481:485"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Is <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection enabled and URL valid?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:434:460"
%%   node1 -->|"Yes"| node2["Connect to Kubernetes cluster and listen for messages"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:464:473"
%%   node1 -->|"No"| node5["Do not connect"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:458:460"
%%   node2 --> node3["Process each incoming message and handle errors"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:437:452"
%%   node2 -->|"On cleanup"| node4["Disconnect from cluster"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:481:485"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="416">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="416:4:4" line-data="export function useWebSocket&lt;T&gt;({">`useWebSocket`</SwmToken> preps the connection and handlers, then moves on to managing the connection lifecycle.

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

Back in <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="416:4:4" line-data="export function useWebSocket&lt;T&gt;({">`useWebSocket`</SwmToken>, we connect and clean up the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="474:6:6" line-data="        console.error(&#39;WebSocket connection failed:&#39;, err);">`WebSocket`</SwmToken> as needed.

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

## Cleaning Up Shared <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start: Manage WebSocket connections"]
    click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:604:649"
    node1 --> node2{"Is enabled?"}
    click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:648:648"
    node2 -->|"No"| node10["Do nothing"]
    click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:648:648"
    node2 -->|"Yes"| loop1
    
    subgraph loop1["For each endpoint in connections"]
      node3["Connect and store disconnect callback"]
      click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:604:642"
      node3 --> node4{"Was hook unmounted during connect?"}
      click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:608:612"
      node4 -->|"Yes"| node5["Close connection and clean up"]
      click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:609:611"
      node4 -->|"No"| node6["Store active connection"]
      click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:614:614"
      node6 --> node7["On cleanup: Remove listener"]
      click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:621:627"
      node7 --> node8{"Are there listeners left?"}
      click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:630:638"
      node8 -->|"No"| node9["Close connection and clean up"]
      click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:631:637"
    end
    loop1 --> node11["On unmount: Call all disconnect callbacks"]
    click node11 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:644:647"
    node10 -->|"If not enabled"| node11
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Start: Manage <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections"]
%%     click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:604:649"
%%     node1 --> node2{"Is enabled?"}
%%     click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:648:648"
%%     node2 -->|"No"| node10["Do nothing"]
%%     click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:648:648"
%%     node2 -->|"Yes"| loop1
%%     
%%     subgraph loop1["For each endpoint in connections"]
%%       node3["Connect and store disconnect callback"]
%%       click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:604:642"
%%       node3 --> node4{"Was hook unmounted during connect?"}
%%       click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:608:612"
%%       node4 -->|"Yes"| node5["Close connection and clean up"]
%%       click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:609:611"
%%       node4 -->|"No"| node6["Store active connection"]
%%       click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:614:614"
%%       node6 --> node7["On cleanup: Remove listener"]
%%       click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:621:627"
%%       node7 --> node8{"Are there listeners left?"}
%%       click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:630:638"
%%       node8 -->|"No"| node9["Close connection and clean up"]
%%       click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:631:637"
%%     end
%%     loop1 --> node11["On unmount: Call all disconnect callbacks"]
%%     click node11 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:644:647"
%%     node10 -->|"If not enabled"| node11
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="604">

---

Back in <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="357:1:1" line-data="  useWebSockets&lt;KubeListUpdateEvent&lt;K&gt;&gt;({">`useWebSockets`</SwmToken>, after opening sockets, we return cleanup functions that remove listeners and close sockets if no one is listening. This keeps resource usage tight and avoids leaks.

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

# Establishing and Tracking Connections

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Request to connect for cluster+URL"] --> node2{"Is there already a connection for this cluster+URL?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:591:594"
  node2 -->|"No"| node3["Add listener, mark as pending, open WebSocket connection"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:594:599"
  node3 --> node4{"Is connection still needed when opened?"}
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:599:614"
  node4 -->|"No"| node5["Close connection and clean up"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:608:611"
  node4 -->|"Yes"| node6["Connection established"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:614:615"
  node2 -->|"Yes"| node6
  node5 --> node7["Return cleanup function"]
  node6 --> node7["Return cleanup function"]
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:621:639"
  node7 --> node8["Cleanup: Remove listener for cluster+URL"]
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:624:626"
  node8 --> node9{"Are any listeners left for cluster+URL?"}
  click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:630:638"
  node9 -->|"No"| node10["Close and remove connection"]
  click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:631:637"
  node9 -->|"Yes"| node11["Keep connection open"]
  click node11 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:638:639"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Request to connect for cluster+URL"] --> node2{"Is there already a connection for this cluster+URL?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:591:594"
%%   node2 -->|"No"| node3["Add listener, mark as pending, open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:594:599"
%%   node3 --> node4{"Is connection still needed when opened?"}
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:599:614"
%%   node4 -->|"No"| node5["Close connection and clean up"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:608:611"
%%   node4 -->|"Yes"| node6["Connection established"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:614:615"
%%   node2 -->|"Yes"| node6
%%   node5 --> node7["Return cleanup function"]
%%   node6 --> node7["Return cleanup function"]
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:621:639"
%%   node7 --> node8["Cleanup: Remove listener for cluster+URL"]
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:624:626"
%%   node8 --> node9{"Are any listeners left for cluster+URL?"}
%%   click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:630:638"
%%   node9 -->|"No"| node10["Close and remove connection"]
%%   click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:631:637"
%%   node9 -->|"Yes"| node11["Keep connection open"]
%%   click node11 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:638:639"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="591">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="591:3:3" line-data="    function connect({ cluster, url, onMessage }: WebSocketConnectionRequest&lt;T&gt;) {">`connect`</SwmToken>, we build a unique key for each connection, add listeners, and mark sockets as 'pending' while opening the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="601:6:6" line-data="        let ws: WebSocket | undefined;">`WebSocket`</SwmToken>. This avoids duplicate connections and lets us track listeners for cleanup.

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

Back in the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="423:17:17" line-data="  /** Function that returns the WebSocket URL to connect to */">`connect`</SwmToken> function, after opening the socket, we return a cleanup function that removes listeners and closes the socket if no one is left listening. This keeps things tidy and avoids leaks.

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
