---
title: Real-Time Kubernetes Resource List Updates
---
This document explains how the system keeps Kubernetes resource lists up-to-date for users. By monitoring specified clusters and namespaces, the system ensures that any changes to resources are reflected in real time. The process chooses the most efficient strategy for managing connections and updates.

# Deciding <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Strategy

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="127">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="127:4:4" line-data="export function useWatchKubeObjectLists&lt;K extends KubeObject&gt;({">`useWatchKubeObjectLists`</SwmToken>, we start by checking if multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> support is enabled. If it is, we delegate to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken> to handle multiple resource lists over a single connection. This step is necessary to optimize resource usage and avoid opening redundant connections for each list.

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

## Setting Up Multiplexed Listeners

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="170">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="170:2:2" line-data="function useWatchKubeObjectListsMultiplexed&lt;K extends KubeObject&gt;({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, we prep the connections and handlers for multiplexed <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="203:5:5" line-data="      // Construct WebSocket URL with current parameters">`WebSocket`</SwmToken> updates. We track resource versions to avoid duplicate updates and set up stable <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="190:9:9" line-data="  // Create stable connection URLs for each list">`URLs`</SwmToken> and handlers. Next, we need to call into the Electron main process to handle plugin updates and cache progress, since that's where the actual update logic and state management lives.

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

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start plugin update"] --> node2{"Is plugin name provided?"}
  click node1 openCode "app/electron/main.ts:365:366"
  node2 -->|"No"| node3["Report error: 'Plugin Name is required' and stop"]
  click node2 openCode "app/electron/main.ts:367:373"
  click node3 openCode "app/electron/main.ts:368:373"
  node2 -->|"Yes"| node4["Track update progress and initiate plugin update"]
  click node4 openCode "app/electron/main.ts:375:391"
  node4 --> node5["Report progress updates to user"]
  click node5 openCode "app/electron/main.ts:387:389"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start plugin update"] --> node2{"Is plugin name provided?"}
%%   click node1 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:365:366"
%%   node2 -->|"No"| node3["Report error: 'Plugin Name is required' and stop"]
%%   click node2 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:367:373"
%%   click node3 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:368:373"
%%   node2 -->|"Yes"| node4["Track update progress and initiate plugin update"]
%%   click node4 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:375:391"
%%   node4 --> node5["Report progress updates to user"]
%%   click node5 openCode "<SwmPath>[app/electron/main.ts](app/electron/main.ts)</SwmPath>:387:389"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/app/electron/main.ts" line="365">

---

<SwmToken path="app/electron/main.ts" pos="365:3:3" line-data="  private handleUpdate(eventData: Action, updateCache: (progress: ProgressResp) =&gt; void) {">`handleUpdate`</SwmToken> checks for required fields, sets an initial cache state with a fixed progress value, and creates an <SwmToken path="app/electron/main.ts" pos="375:9:9" line-data="    const controller = new AbortController();">`AbortController`</SwmToken> for cancellation. It then calls <SwmToken path="app/electron/main.ts" pos="383:1:3" line-data="    PluginManager.update(">`PluginManager.update`</SwmToken> to actually perform the plugin update, passing along a progress callback and the abort signal. The next step is needed because the update logic and file operations are handled in <SwmToken path="app/electron/main.ts" pos="52:6:8" line-data="} from &#39;./plugin-management&#39;;">`plugin-management`</SwmToken>, not here.

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

### Updating and Downloading Plugin Files

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Check installed plugins"] --> node2{"Is plugin installed?"}
  click node1 openCode "app/electron/plugin-management.ts:248:249"
  node2 -->|"No"| node3["Report error: Plugin not found"]
  click node2 openCode "app/electron/plugin-management.ts:249:251"
  click node3 openCode "app/electron/plugin-management.ts:299:300"
  node2 -->|"Yes"| node4{"Is plugin found?"}
  click node4 openCode "app/electron/plugin-management.ts:252:255"
  node4 -->|"No"| node3
  node4 -->|"Yes"| node5["Get current and latest plugin version"]
  click node5 openCode "app/electron/plugin-management.ts:260:265"
  node5 --> node6{"Is update available?"}
  click node6 openCode "app/electron/plugin-management.ts:267:269"
  node6 -->|"No"| node7["Report error: No updates available"]
  click node7 openCode "app/electron/plugin-management.ts:299:300"
  node6 -->|"Yes"| node8["Download and extract latest plugin"]
  click node8 openCode "app/electron/plugin-management.ts:272:277"
  node8 --> node9{"Does destination folder exist?"}
  click node9 openCode "app/electron/plugin-management.ts:282:284"
  node9 -->|"No"| node10["Create destination folder"]
  click node10 openCode "app/electron/plugin-management.ts:283:284"
  node9 -->|"Yes"| node11["Remove old plugin files"]
  click node11 openCode "app/electron/plugin-management.ts:287:288"
  node10 --> node11
  node11 --> node12["Move updated plugin files"]
  click node12 openCode "app/electron/plugin-management.ts:293:293"
  node12 --> node13{"Was progressCallback provided?"}
  click node13 openCode "app/electron/plugin-management.ts:294:296"
  node13 -->|"Yes"| node14["Report success"]
  click node14 openCode "app/electron/plugin-management.ts:295:296"
  node13 -->|"No"| node15["Finish update"]
  click node15 openCode "app/electron/plugin-management.ts:297:303"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Check installed plugins"] --> node2{"Is plugin installed?"}
%%   click node1 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:248:249"
%%   node2 -->|"No"| node3["Report error: Plugin not found"]
%%   click node2 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:249:251"
%%   click node3 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:299:300"
%%   node2 -->|"Yes"| node4{"Is plugin found?"}
%%   click node4 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:252:255"
%%   node4 -->|"No"| node3
%%   node4 -->|"Yes"| node5["Get current and latest plugin version"]
%%   click node5 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:260:265"
%%   node5 --> node6{"Is update available?"}
%%   click node6 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:267:269"
%%   node6 -->|"No"| node7["Report error: No updates available"]
%%   click node7 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:299:300"
%%   node6 -->|"Yes"| node8["Download and extract latest plugin"]
%%   click node8 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:272:277"
%%   node8 --> node9{"Does destination folder exist?"}
%%   click node9 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:282:284"
%%   node9 -->|"No"| node10["Create destination folder"]
%%   click node10 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:283:284"
%%   node9 -->|"Yes"| node11["Remove old plugin files"]
%%   click node11 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:287:288"
%%   node10 --> node11
%%   node11 --> node12["Move updated plugin files"]
%%   click node12 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:293:293"
%%   node12 --> node13{"Was <SwmToken path="app/electron/plugin-management.ts" pos="243:1:1" line-data="    progressCallback: null | ProgressCallback = null,">`progressCallback`</SwmToken> provided?"}
%%   click node13 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:294:296"
%%   node13 -->|"Yes"| node14["Report success"]
%%   click node14 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:295:296"
%%   node13 -->|"No"| node15["Finish update"]
%%   click node15 openCode "<SwmPath>[app/electron/plugin-management.ts](app/electron/plugin-management.ts)</SwmPath>:297:303"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/app/electron/plugin-management.ts" line="239">

---

In <SwmToken path="app/electron/main.ts" pos="383:1:3" line-data="    PluginManager.update(">`PluginManager.update`</SwmToken>, we list installed plugins, check the current version from <SwmPath>[package.json](package.json)</SwmPath>, fetch the latest info, and compare versions. If an update is needed, we call <SwmToken path="app/electron/plugin-management.ts" pos="272:14:14" line-data="      const [_, tempFolder] = await downloadExtractArchive(">`downloadExtractArchive`</SwmToken> to get the new files. This step is necessary to actually fetch and prepare the updated plugin before replacing the old one.

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

<SwmToken path="app/electron/plugin-management.ts" pos="469:4:4" line-data="async function downloadExtractArchive(">`downloadExtractArchive`</SwmToken> validates the plugin name, checks Headlamp compatibility, creates a temp directory, downloads and extracts the plugin archive and extra files, then updates <SwmPath>[package.json](package.json)</SwmPath> with metadata and a management flag. Progress and cancellation are supported throughout. This prepares the plugin for installation or update.

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

We just returned from <SwmToken path="app/electron/plugin-management.ts" pos="272:14:14" line-data="      const [_, tempFolder] = await downloadExtractArchive(">`downloadExtractArchive`</SwmToken> in <SwmToken path="app/electron/main.ts" pos="52:6:8" line-data="} from &#39;./plugin-management&#39;;">`plugin-management`</SwmToken>, so now we remove the old plugin folder, create a new one, and move the downloaded files into place. The update is finalized and the progress callback is called with success or error.

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

### Cleaning Up Multiplexed Subscriptions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["WebSocket subscription fails"] --> node2{"Is retry count in URL < 3?"}
    click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:281:281"
    click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:279:283"
    node2 -->|"Yes"| node3["Log error and increment retry count in URL"]
    click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:281:283"
    node2 -->|"No"| node4["Stop retrying"]
    click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:279:283"
    node3 --> node5["Wait for next subscription attempt"]
    click node5 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:284:284"
    node4 --> node6["Cleanup subscriptions when effect re-runs or unmounts"]
    click node6 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:288:291"
    node5 --> node6
    subgraph loop1["For each active subscription"]
      node6 --> node7["Run cleanup"]
      click node7 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:290:290"
    end
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> subscription fails"] --> node2{"Is retry count in URL < 3?"}
%%     click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:281:281"
%%     click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:279:283"
%%     node2 -->|"Yes"| node3["Log error and increment retry count in URL"]
%%     click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:281:283"
%%     node2 -->|"No"| node4["Stop retrying"]
%%     click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:279:283"
%%     node3 --> node5["Wait for next subscription attempt"]
%%     click node5 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:284:284"
%%     node4 --> node6["Cleanup subscriptions when effect <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="288:11:13" line-data="    // Cleanup subscriptions when effect re-runs or unmounts">`re-runs`</SwmToken> or unmounts"]
%%     click node6 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:288:291"
%%     node5 --> node6
%%     subgraph loop1["For each active subscription"]
%%       node6 --> node7["Run cleanup"]
%%       click node7 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:290:290"
%%     end
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="277">

---

After <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="292:11:11" line-data="  }, [connections, endpoint, handleUpdate]);">`handleUpdate`</SwmToken>, <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken> cleans up subscriptions to avoid resource leaks.

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

## Fallback to Legacy Watchers

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="150">

---

We just returned from <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="143:3:3" line-data="    return useWatchKubeObjectListsMultiplexed({">`useWatchKubeObjectListsMultiplexed`</SwmToken>, so if multiplexing isn't enabled, <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="127:4:4" line-data="export function useWatchKubeObjectLists&lt;K extends KubeObject&gt;({">`useWatchKubeObjectLists`</SwmToken> falls back to calling <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="150:3:3" line-data="    return useWatchKubeObjectListsLegacy({">`useWatchKubeObjectListsLegacy`</SwmToken> to handle resource list updates using the older method.

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

# Setting Up Legacy Listeners

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Start monitoring Kubernetes resources"] --> node2{"Is endpoint provided?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:303:361"
  node2 -->|"No"| node5["No monitoring set up"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:321:322"
  node2 -->|"Yes"| node3["Prepare monitoring connections for each frontend/…/components/namespace in lists using query parameters"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:323:354"
  subgraph loop1["For each entry in lists"]
    node3 --> node4["Create connection with update handler (keeps resource list up to date)"]
    click node4 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:330:352"
  end
  node4 --> node6["Establish WebSocket monitoring for all connections"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:357:360"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Start monitoring Kubernetes resources"] --> node2{"Is endpoint provided?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:303:361"
%%   node2 -->|"No"| node5["No monitoring set up"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:321:322"
%%   node2 -->|"Yes"| node3["Prepare monitoring connections for each <SwmPath>[frontend/…/components/namespace/](frontend/src/components/namespace/)</SwmPath> in lists using query parameters"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:323:354"
%%   subgraph loop1["For each entry in lists"]
%%     node3 --> node4["Create connection with update handler (keeps resource list up to date)"]
%%     click node4 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:330:352"
%%   end
%%   node4 --> node6["Establish <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> monitoring for all connections"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/useKubeObjectList.ts](frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)</SwmPath>:357:360"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" line="303">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="303:2:2" line-data="function useWatchKubeObjectListsLegacy&lt;K extends KubeObject&gt;({">`useWatchKubeObjectListsLegacy`</SwmToken> sets up connection configs for each resource list and defines how updates are handled. It then calls <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="357:1:1" line-data="  useWebSockets&lt;KubeListUpdateEvent&lt;K&gt;&gt;({">`useWebSockets`</SwmToken> to actually manage the connections and message handling.

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
  node1 -->|"Yes"| loop1
  node1 -->|"No"| node5["WebSocket Connection Cleanup"]
  

  subgraph loop1["For each connection request"]
    node2{"Is connection already open?"}
    
    node2 -->|"No"| node3["Opening Cluster-Aware WebSocket"]
    
    node2 -->|"Yes"| node4["WebSocket Connection Cleanup"]
    
  end
  loop1 --> node5
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node3 goToHeading "Opening Cluster-Aware WebSocket"
node3:::HeadingStyle
click node2 goToHeading "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:135 Connection Cleanup"
node2:::HeadingStyle
click node4 goToHeading "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:135 Connection Cleanup"
node4:::HeadingStyle
click node5 goToHeading "frontend/src/lib/k8s/api/v2/useKubeObjectList.ts:135 Connection Cleanup"
node5:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Are <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connections enabled?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:585:586"
%%   node1 -->|"Yes"| loop1
%%   node1 -->|"No"| node5["<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Cleanup"]
%%   
%% 
%%   subgraph loop1["For each connection request"]
%%     node2{"Is connection already open?"}
%%     
%%     node2 -->|"No"| node3["Opening Cluster-Aware <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>"]
%%     
%%     node2 -->|"Yes"| node4["<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Cleanup"]
%%     
%%   end
%%   loop1 --> node5
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node3 goToHeading "Opening Cluster-Aware <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>"
%% node3:::HeadingStyle
%% click node2 goToHeading "<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Cleanup"
%% node2:::HeadingStyle
%% click node4 goToHeading "<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Cleanup"
%% node4:::HeadingStyle
%% click node5 goToHeading "<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Cleanup"
%% node5:::HeadingStyle
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="566">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="566:4:4" line-data="export function useWebSockets&lt;T&gt;({">`useWebSockets`</SwmToken>, we use global maps to manage <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="576:15:15" line-data="   * Any additional protocols to include in WebSocket connection">`WebSocket`</SwmToken> connections and listeners, keyed by cluster+url. The 'pending' marker prevents duplicate connection attempts. Next, we call <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="602:1:1" line-data="        openWebSocket(url, { protocols, type, cluster, onMessage })">`openWebSocket`</SwmToken> to actually establish the connection and set up message handling.

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

## Opening Cluster-Aware <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken>

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Prepare connection details (cluster, protocols)"] --> node2{"Is a cluster specified?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:529:532"
  node2 -->|"Yes"| node3["Connect to specified cluster"]
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:532:545"
  node2 -->|"No"| node5["Open WebSocket connection"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:532:545"
  node3 --> node4{"Is user authorization available?"}
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:538:541"
  node4 -->|"Yes"| node6["Add user authorization to protocols"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:539:541"
  node4 -->|"No"| node5
  node6 --> node5
  node5["Open WebSocket connection"] --> node7["Return WebSocket connection"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:547:557"
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:557:558"
  node5 -.-> node8{"When a message is received"}
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:549:552"
  node8 -->|"type = JSON"| node9["Parse message as JSON and call onMessage"]
  click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:550:551"
  node8 -->|"type = binary"| node10["Call onMessage with binary data"]
  click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:551:551"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Prepare connection details (cluster, protocols)"] --> node2{"Is a cluster specified?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:529:532"
%%   node2 -->|"Yes"| node3["Connect to specified cluster"]
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:532:545"
%%   node2 -->|"No"| node5["Open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:532:545"
%%   node3 --> node4{"Is user authorization available?"}
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:538:541"
%%   node4 -->|"Yes"| node6["Add user authorization to protocols"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:539:541"
%%   node4 -->|"No"| node5
%%   node6 --> node5
%%   node5["Open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"] --> node7["Return <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:547:557"
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:557:558"
%%   node5 -.-> node8{"When a message is received"}
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:549:552"
%%   node8 -->|"type = JSON"| node9["Parse message as JSON and call <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="333:1:1" line-data="        onMessage(update: KubeListUpdateEvent&lt;K&gt;) {">`onMessage`</SwmToken>"]
%%   click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:550:551"
%%   node8 -->|"type = binary"| node10["Call <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="333:1:1" line-data="        onMessage(update: KubeListUpdateEvent&lt;K&gt;) {">`onMessage`</SwmToken> with binary data"]
%%   click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:551:551"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="503">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="503:6:6" line-data="export async function openWebSocket&lt;T&gt;(">`openWebSocket`</SwmToken> builds the URL and protocols for the connection, adding cluster and authorization info if needed. It sets up message and error handlers, then returns the socket. Next, we call into hooks to process incoming messages.

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

## Processing Incoming Updates

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/hooks.ts" line="152">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="333:1:1" line-data="        onMessage(update: KubeListUpdateEvent&lt;K&gt;) {">`onMessage`</SwmToken> in hooks processes incoming updates, updating the cache for non-ADDED events using the <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="128:1:1" line-data="  kubeObjectClass,">`kubeObjectClass`</SwmToken>. Next, we go back to the <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> logic to manage connection lifecycle and cleanup.

```typescript
    onMessage(update: KubeListUpdateEvent<K>) {
      if (update.type !== 'ADDED' && update.object) {
        client.setQueryData(queryKey, new kubeObjectClass(update.object));
      }
    },
```

---

</SwmSnippet>

## Single <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Lifecycle

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Should WebSocket connection be active?"]
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:434:435"
  node1 --> node2{"Is enabled true and URL valid?"}
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:458:460"
  node2 -->|"Yes"| node3["Connect to WebSocket for cluster"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:464:473"
  node3 --> node4["Subscribe to real-time updates"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:464:473"
  node4 --> node5["Handle incoming messages via callback"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:437:449"
  node5 --> node7["Cleanup connection when closed"]
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:481:485"
  node4 --> node6["Handle errors via callback"]
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:450:452"
  node2 -->|"No"| node8["No connection established"]
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:459:460"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Should <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection be active?"]
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:434:435"
%%   node1 --> node2{"Is enabled true and URL valid?"}
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:458:460"
%%   node2 -->|"Yes"| node3["Connect to <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> for cluster"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:464:473"
%%   node3 --> node4["Subscribe to real-time updates"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:464:473"
%%   node4 --> node5["Handle incoming messages via callback"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:437:449"
%%   node5 --> node7["Cleanup connection when closed"]
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:481:485"
%%   node4 --> node6["Handle errors via callback"]
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:450:452"
%%   node2 -->|"No"| node8["No connection established"]
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:459:460"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="416">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="416:4:4" line-data="export function useWebSocket&lt;T&gt;({">`useWebSocket`</SwmToken> sets up the connection URL and message handler, catching and logging errors. It then calls hooks to process incoming messages and manage cache updates.

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

After hooks, <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="416:4:4" line-data="export function useWebSocket&lt;T&gt;({">`useWebSocket`</SwmToken> manages the subscription lifecycle and cleanup.

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

## <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> Connection Cleanup

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start WebSocket management"]
    click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:604:649"
    subgraph loop1["For each endpoint"]
      node1 --> node2["Set up WebSocket connection"]
      click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:604:615"
      node2 --> node3{"Is hook still mounted?"}
      click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:608:612"
      node3 -->|"No"| node4["Close connection and clean up"]
      click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:609:611"
      node3 -->|"Yes"| node5["Add connection to active sockets"]
      click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:614:615"
      node5 --> node6["Manage listeners"]
      click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:625:627"
      node6 --> node7{"Are there listeners left?"}
      click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:630:638"
      node7 -->|"No"| node8["Close connection and clean up"]
      click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:631:637"
      node7 -->|"Yes"| node9["Keep connection open"]
      click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:639:639"
    end
    subgraph loop2["On unmount, for each endpoint"]
      node10["Invoke disconnect callback"]
      click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:645:647"
    end

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Start <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> management"]
%%     click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:604:649"
%%     subgraph loop1["For each endpoint"]
%%       node1 --> node2["Set up <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%       click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:604:615"
%%       node2 --> node3{"Is hook still mounted?"}
%%       click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:608:612"
%%       node3 -->|"No"| node4["Close connection and clean up"]
%%       click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:609:611"
%%       node3 -->|"Yes"| node5["Add connection to active sockets"]
%%       click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:614:615"
%%       node5 --> node6["Manage listeners"]
%%       click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:625:627"
%%       node6 --> node7{"Are there listeners left?"}
%%       click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:630:638"
%%       node7 -->|"No"| node8["Close connection and clean up"]
%%       click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:631:637"
%%       node7 -->|"Yes"| node9["Keep connection open"]
%%       click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:639:639"
%%     end
%%     subgraph loop2["On unmount, for each endpoint"]
%%       node10["Invoke disconnect callback"]
%%       click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:645:647"
%%     end
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="604">

---

We just returned from the socket opening logic, so at the end of <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="357:1:1" line-data="  useWebSockets&lt;KubeListUpdateEvent&lt;K&gt;&gt;({">`useWebSockets`</SwmToken>, we handle cleanup for all connections and listeners. This prevents resource leaks and duplicate connections by closing sockets when no listeners remain.

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
  node1["Request WebSocket connection for cluster+url"] --> node2{"Is there already a socket for this connectionKey?"}
  click node1 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:591:592"
  click node2 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:594:594"
  node2 -->|"No"| node3["Add listener for connectionKey"]
  click node3 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:596:596"
  node3 --> node4["Mark socket as pending"]
  click node4 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:599:599"
  node4 --> node5["Open WebSocket connection"]
  click node5 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:602:603"
  node5 --> node6{"Was hook unmounted before connection established?"}
  click node6 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:608:608"
  node6 -->|"Yes"| node7["Close socket and remove from sockets"]
  click node7 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:609:611"
  node6 -->|"No"| node8["Store socket in sockets"]
  click node8 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:614:614"
  node2 -->|"Yes"| node9["Add listener for connectionKey"]
  click node9 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:596:596"
  node10["Return cleanup function"]
  click node10 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:621:639"
  node10 --> node11["Remove listener for connectionKey"]
  click node11 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:625:626"
  node11 --> node12{"Are there any listeners left for connectionKey?"}
  click node12 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:630:630"
  node12 -->|"No"| node13{"Is socket established?"}
  click node13 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:631:633"
  node13 -->|"Yes"| node14["Close socket"]
  click node14 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:634:635"
  node13 -->|"No"| node15["Remove socket from sockets"]
  click node15 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:636:637"
  node12 -->|"Yes"| node16["Keep connection open"]
  click node16 openCode "frontend/src/lib/k8s/api/v2/webSocket.ts:638:638"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Request <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection for cluster+url"] --> node2{"Is there already a socket for this <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken>?"}
%%   click node1 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:591:592"
%%   click node2 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:594:594"
%%   node2 -->|"No"| node3["Add listener for <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken>"]
%%   click node3 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:596:596"
%%   node3 --> node4["Mark socket as pending"]
%%   click node4 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:599:599"
%%   node4 --> node5["Open <SwmToken path="frontend/src/lib/k8s/api/v2/useKubeObjectList.ts" pos="135:11:11" line-data="  /** Query parameters for the WebSocket connection URL */">`WebSocket`</SwmToken> connection"]
%%   click node5 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:602:603"
%%   node5 --> node6{"Was hook unmounted before connection established?"}
%%   click node6 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:608:608"
%%   node6 -->|"Yes"| node7["Close socket and remove from sockets"]
%%   click node7 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:609:611"
%%   node6 -->|"No"| node8["Store socket in sockets"]
%%   click node8 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:614:614"
%%   node2 -->|"Yes"| node9["Add listener for <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken>"]
%%   click node9 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:596:596"
%%   node10["Return cleanup function"]
%%   click node10 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:621:639"
%%   node10 --> node11["Remove listener for <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken>"]
%%   click node11 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:625:626"
%%   node11 --> node12{"Are there any listeners left for <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="592:3:3" line-data="      const connectionKey = cluster + url;">`connectionKey`</SwmToken>?"}
%%   click node12 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:630:630"
%%   node12 -->|"No"| node13{"Is socket established?"}
%%   click node13 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:631:633"
%%   node13 -->|"Yes"| node14["Close socket"]
%%   click node14 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:634:635"
%%   node13 -->|"No"| node15["Remove socket from sockets"]
%%   click node15 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:636:637"
%%   node12 -->|"Yes"| node16["Keep connection open"]
%%   click node16 openCode "<SwmPath>[frontend/…/v2/webSocket.ts](frontend/src/lib/k8s/api/v2/webSocket.ts)</SwmPath>:638:638"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="591">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="591:3:3" line-data="    function connect({ cluster, url, onMessage }: WebSocketConnectionRequest&lt;T&gt;) {">`connect`</SwmToken> builds a unique key for each connection, manages the socket and listeners maps, and only opens a new socket if one doesn't exist or isn't pending. Next, it calls <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="602:1:1" line-data="        openWebSocket(url, { protocols, type, cluster, onMessage })">`openWebSocket`</SwmToken> to actually establish the connection.

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

After opening the socket, <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="423:17:17" line-data="  /** Function that returns the WebSocket URL to connect to */">`connect`</SwmToken> cleans up listeners and sockets as needed.

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
