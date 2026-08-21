# SA-MP GPS 插件

[![sampctl](https://img.shields.io/badge/sampctl-samp--gps--plugin-2f2f2f.svg?style=for-the-badge)](https://github.com/kristoisberg/samp-gps-plugin)

[English](README.md) | **简体中文**

**注意：** 本仓库已不再积极维护。如果您希望继续开发这个项目，请创建仓库的 fork，并在那里发布后续版本。

该插件提供了一种访问和操作圣安地列斯（San Andreas）地图节点数据以及在这些节点之间寻路的方式。它旨在成为 RouteConnector 的现代化、简洁的替代品。插件使用 A* 算法的简单实现进行寻路。从地图最左上角的节点寻路到最右下角的节点（由 684 个节点组成）只需几毫秒。


### 相比 RouteConnector 的优势

* **更安全的 API** - 与 RouteConnector 不同，本插件不会把节点数组作为寻路结果返回给你。取而代之的是，它返回一个路径的 ID，供你之后使用。每个函数（除了 `IsValidMapNode`、`IsValidPath` 和 `GetHighestMapNodeID`）都会返回一个错误码，真正的结果通过引用（by reference）传出。
* **兼容性** - RouteConnector 与 YSI 的某些部分存在兼容性问题，会导致它调用错误的公共函数而不是真正的 `GPS_WhenRouteIsCalculated` 回调。本插件允许你调用自定义回调并向其传递参数。此外，RouteConnector 使用 Intel Threading Building Blocks 进行多线程处理，这在 Linux 服务器上引发了许多兼容性（以及 PEBCAK）问题。本插件使用 `std::thread` 进行多线程处理，没有任何依赖。本插件还兼容 PawnPlus，开箱即用地支持异步寻路。
* **性能** - 我虽然没有做过基准测试，但即使是旧版本，用户也声称该插件比 RouteConnector 快好几倍。1.2.0 版本中对算法的修复使其比之前快了约 30 倍。


## 安装

只需将其安装到你的项目中：

```bash
sampctl package install kristoisberg/samp-gps-plugin
```

在你的代码中引入该库并开始使用：

```pawn
#include <GPS>
```


## API


### 函数

`CreateMapNode(Float:x, Float:y, Float:z, &MapNode:nodeid)`

* 向地图添加一个节点，并将其 ID 传给 `nodeid`。

`DestroyMapNode(MapNode:nodeid)`

* 如果指定的地图节点有效，则返回 `GPS_ERROR_NONE` 并尝试销毁它，否则返回 `GPS_ERROR_INVALID_NODE`。如果该节点不属于任何路径，它会立即被销毁；否则，它将等到包含它的所有路径都被销毁后再销毁，在此之前它会从寻路和其他若干功能中被排除。

`bool:IsValidMapNode(MapNode:nodeid)`

* 返回具有指定 ID 的地图节点是否有效。

`GetMapNodePos(MapNode:nodeid, &Float:x, &Float:y, &Float:z)`

* 如果指定的地图节点有效，则返回 `GPS_ERROR_NONE` 并将其坐标传给 `x`、`y` 和 `z`，否则返回 `GPS_ERROR_INVALID_NODE`。

`CreateConnection(MapNode:source, MapNode:target, &Connection:connectionid)`

* 如果指定的两个地图节点都有效，则返回 `GPS_ERROR_NONE`，创建一条从 `source` 到 `target` 的连接，并将其 ID 传给 `connectionid`，否则返回 `GPS_ERROR_INVALID_NODE`。**注意：** 连接不是双向的，可能需要分别添加两个方向的连接。

`DestroyConnection(Connection:connectionid)`

* 如果指定的连接有效，则返回 `GPS_ERROR_NONE` 并销毁它，否则返回 `GPS_ERROR_INVALID_CONNECTION`。

`GetConnectionSource(Connection:connectionid, &MapNode:nodeid)`

* 如果指定的连接有效，则返回 `GPS_ERROR_NONE` 并将其源节点的 ID 传给 `nodeid`，否则返回 `GPS_ERROR_INVALID_CONNECTION`。

`GetConnectionTarget(Connection:connectionid, &MapNode:nodeid)`

* 如果指定的连接有效，则返回 `GPS_ERROR_NONE` 并将其目标节点的 ID 传给 `nodeid`，否则返回 `GPS_ERROR_INVALID_CONNECTION`。

`GetMapNodeConnectionCount(MapNode:nodeid, &count)`

* 如果指定的地图节点有效，则返回 `GPS_ERROR_NONE` 并将其连接数量传给 `count`，否则返回 `GPS_ERROR_INVALID_NODE`。如果 `count` 大于 2，则该节点是一个交叉路口。

`GetMapNodeConnection(MapNode:nodeid, index, &Connection:connectionid)`

* 如果指定的地图节点有效且它存在指定索引的连接，则返回 `GPS_ERROR_NONE` 并将该连接的 ID 传给 `connectionid`，否则根据错误情况返回 `GPS_ERROR_INVALID_NODE` 或 `GPS_ERROR_INVALID_CONNECTION`。

`GetConnectionBetweenMapNodes(MapNode:source, MapNode:target, &Connection:connectionid)`

* 如果指定的两个地图节点都有效，则尝试查找一条从 `source` 到 `target` 的连接。如果找到连接，则返回 `GPS_ERROR_NONE` 并将该连接的 ID 传给 `connectionid`，否则返回 `GPS_ERROR_INVALID_CONNECTION`。如果指定的任一地图节点无效，则返回 `GPS_ERROR_INVALID_NODE`。

`GetDistanceBetweenMapNodes(MapNode:first, MapNode:second, &Float:distance)`

* 如果指定的两个地图节点都有效，则返回 `GPS_ERROR_NONE` 并将它们之间的距离传给 `distance`，否则返回 `GPS_ERROR_INVALID_NODE`。

`GetAngleBetweenMapNodes(MapNode:first, MapNode:second, &Float:angle)`

* 如果指定的两个地图节点都有效，则返回 `GPS_ERROR_NONE` 并将它们之间的角度传给 `angle`，否则返回 `GPS_ERROR_INVALID_NODE`。

`GetMapNodeDistanceFromPoint(MapNode:nodeid, Float:x, Float:y, Float:z, &Float:distance)`

* 如果指定的地图节点有效，则返回 `GPS_ERROR_NONE` 并将该地图节点到指定坐标的距离传给 `distance`，否则返回 `GPS_ERROR_INVALID_NODE`。

`GetMapNodeAngleFromPoint(MapNode:nodeid, Float:x, Float:y, &Float:angle)`

* 如果指定的地图节点有效，则返回 `GPS_ERROR_NONE` 并将该地图节点相对于指定坐标的角度传给 `angle`，否则返回 `GPS_ERROR_INVALID_NODE`。

`GetClosestMapNodeToPoint(Float:x, Float:y, Float:z, &MapNode:nodeid, MapNode:ignorednode = INVALID_MAP_NODE_ID)`

* 将距离指定坐标最近的地图节点的 ID 传给 `nodeid`。如果指定了 `ignorednode` 且它正好是距离该坐标最近的节点，则它会被忽略，并将第二近的节点的 ID 传给 `nodeid`。如果不存在任何节点，则返回 `GPS_ERROR_INVALID_NODE`，否则返回 `GPS_ERROR_NONE`。

`GetHighestMapNodeID()`

* 返回 ID 最大的地图节点的 ID。可用于遍历等用途。

`GetRandomMapNode(&MapNode:nodeid)`

* 使用梅森旋转算法（Mersenne Twister）随机选出一个地图节点，并将其 ID 传给 `nodeid`。如果不存在任何地图节点，则返回 `GPS_ERROR_INVALID_NODE`，否则返回 `GPS_ERROR_NONE`。

`SaveMapNodesToFile(const filename[])`

* 将所有现有节点及其连接保存到指定名称的文件中。

`FindPath(MapNode:source, MapNode:target, &Path:pathid)`

* 如果指定的两个地图节点都有效，则返回 `GPS_ERROR_NONE` 并尝试查找一条从 `source` 到 `target` 的路径，将其 ID 传给 `pathid`，否则返回 `GPS_ERROR_INVALID_NODE`。如果寻路失败，则返回 `GPS_ERROR_INVALID_PATH`。

`FindPathThreaded(MapNode:source, MapNode:target, const callback[], const format[] = "", {Float, _}:...)`

* 如果指定的两个地图节点都有效，则返回 `GPS_ERROR_NONE` 并尝试查找一条从 `source` 到 `target` 的路径。寻路完成后，调用指定的回调，并将路径 ID（如果寻路失败，则可能是 `INVALID_GPS_PATH_ID`）和指定的参数传给它。

`Task:FindPathAsync(MapNode:source, MapNode:target)`

* 暂停当前函数，并在其结束后继续执行。如果寻路因任何原因失败，则会抛出 AMX 错误。仅在 PawnPlus 先于 GPS 被引入时可用。用法见下文。

`bool:IsValidPath(Path:pathid)`

* 返回具有指定 ID 的路径是否有效。

`GetPathSize(Path:pathid, &size)`

* 如果指定的路径有效，则返回 `GPS_ERROR_NONE` 并将其包含的节点数量传给 `size`，否则返回 `GPS_ERROR_INVALID_PATH`。

`GetPathNode(Path:pathid, index, &MapNode:nodeid)`

* 如果指定的路径有效且该索引处存在节点，则返回 `GPS_ERROR_NONE` 并将该索引处的节点 ID 传给 `nodeid`，否则根据错误情况返回 `GPS_ERROR_INVALID_PATH` 或 `GPS_ERROR_INVALID_NODE`。

`GetPathNodeIndex(Path:pathid, MapNode:nodeid, &index)`

* 如果指定的路径有效且指定的地图节点是该路径的一部分，则返回 `GPS_ERROR_NONE` 并将该地图节点的索引传给 `index`，否则根据错误情况返回 `GPS_ERROR_INVALID_PATH` 或 `GPS_ERROR_INVALID_NODE`。

`GetPathLength(Path:pathid, &Float:length)`

* 如果指定的路径有效，则返回 `GPS_ERROR_NONE` 并将路径长度（以米为单位）传给 `length`，否则返回 `GPS_ERROR_INVALID_PATH`。

`DestroyPath(Path:pathid)`

* 如果指定的路径有效，则返回 `GPS_ERROR_NONE` 并销毁该路径，否则返回 `GPS_ERROR_INVALID_PATH`。


### 错误码

* `GPS_ERROR_NONE` - 函数执行成功。
* `GPS_ERROR_INVALID_PARAMS` - 传入函数的参数数量无效。除非插件和 include 的版本不同，否则 PAWN 编译器应该会发现这个问题。
* `GPS_ERROR_INVALID_PATH` - 传入函数的路径 ID 无效，或者线程寻路没有成功。
* `GPS_ERROR_INVALID_NODE` - 传入函数的地图节点 ID/索引无效，或者 `GetClosestMapNodeToPoint` / `GetRandomMapNode` 因为不存在任何地图节点而失败。
* `GPS_ERROR_INVALID_CONNECTION` - 传入函数的连接 ID/索引无效。
* `GPS_ERROR_INTERNAL` - 发生了内部错误 - 线程寻路失败是因为派发线程失败。


## 示例


### 线程寻路

查找一条从玩家位置到 LSPD 大楼的路径。

```pawn
CMD:pathtols(playerid) {
    new Float:x, Float:y, Float:z, MapNode:start;
    GetPlayerPos(playerid, x, y, z);

    if (GetClosestMapNodeToPoint(x, y, z, start) != GPS_ERROR_NONE) {
        return SendClientMessage(playerid, COLOR_RED, "Finding a node near you failed, GPS.dat was not loaded.");
    }

    new MapNode:target;

    if (GetClosestMapNodeToPoint(1258.7352, -2036.7100, 59.4561, target)) { // this is also valid since the value of GPS_ERROR_NONE is 0.
        return SendClientMessage(playerid, COLOR_RED, "Finding a node near LSPD failed, GPS.dat was not loaded.");
    }

    if (FindPathThreaded(start, target, "OnPathToLSFound", "ii", playerid, GetTickCount())) {
        return SendClientMessage(playerid, COLOR_RED, "Pathfinding failed for some reason, you should store this error code and print it out since there are multiple ways it could fail.");
    }

    SendClientMessage(playerid, COLOR_WHITE, "Finding the path...");
    return 1;
}


forward public OnPathToLSFound(Path:pathid, playerid, start_time);
public OnPathToLSFound(Path:pathid, playerid, start_time) {
    if (!IsValidPath(pathid)) {
        return SendClientMessage(playerid, COLOR_RED, "Pathfinding failed!");
    }

    new string[128], size, length;
    GetPathSize(size);
    GetPathLength(length);

    format(string, sizeof(string), "Found a path in %ims. Amount of nodes: %i, length: %fm.", GetTickCount() - start_time, size, length);

    new MapNode:nodeid, Float:x, Float:y, Float:z;

    for (new index; index < size; index++) {
        GetPathNode(pathid, index, nodeid);
        GetMapNodePos(nodeid, x, y, z);
        CreateDynamicPickup(1318, 1, x, y, z);
    }

    DestroyPath(pathid);
    return 1;
}
```


### 异步寻路

如果能在命令内继续处理流程，同时仍然享受线程寻路的好处，会怎样？借助 PawnPlus task 的魔力，你可以做到。

```pawn
CMD:pathtols(playerid) {
    new Float:x, Float:y, Float:z, MapNode:start;
    GetPlayerPos(playerid, x, y, z);

    if (GetClosestMapNodeToPoint(x, y, z, start)) {
        return SendClientMessage(playerid, COLOR_RED, "Finding a node near you failed, GPS.dat was not loaded.");
    }

    new MapNode:target;

    if (GetClosestMapNodeToPoint(1258.7352, -2036.7100, 59.4561, target)) { 
        return SendClientMessage(playerid, COLOR_RED, "Finding a node near LSPD failed, GPS.dat was not loaded.");
    }

    SendClientMessage(playerid, COLOR_WHITE, "Finding the path...");

    new Path:pathid = task_await(FindPathAsync(start, target)); // no error handling here, an AMX error will be thrown instead if the pathfinding fails

    new string[128], size, length;
    GetPathSize(size);
    GetPathLength(length);

    format(string, sizeof(string), "Found a path in %ims. Amount of nodes: %i, length: %fm.", GetTickCount() - start_time, size, length);

    new MapNode:nodeid, Float:x, Float:y, Float:z, index;

    while (!GetPathNode(pathid, index, nodeid)) // also note the alternative method of iterating through path nodes here
        GetMapNodePos(nodeid, x, y, z);
        CreateDynamicPickup(1318, 1, x, y, z);

        index++;
    }

    DestroyPath(pathid);
    return 1;
}
```


## 测试

要测试，只需运行包：

```bash
sampctl package run
```


## 致谢

* kristo - 插件作者。
* Gamer_Z - 原始 RouteConnector 插件的作者，他的插件帮助我理解了 GPS.dat 的结构，并对本插件产生了很大影响；他也是原始 `GPS.dat` 的作者。
* NaS - 随插件分发的修复版 `GPS.dat` 的作者。
* Southclaws、IllidanS4、Hual - 在多个方面给予了我重大帮助（还有其他乐于助人的人，我都很感激）。
