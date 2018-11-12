# snapshotter

### 为什么要有snapshotter?
https://blog.mobyproject.org/where-are-containerds-graph-drivers-145fc9b7255

> These differ from the concept of the graphdriver in that the
> Snapshotter has no knowledge of images or containers. Users simply
> prepare and commit directories. We also avoid the integration between
> graph drivers and the tar format used to represent the changesets.
> 
> The Snapshotter will only provide mount-oriented snapshot access with
> minimal metadata. Serialization, hashing, unpacking, packing and
> mounting are not included in this design, opting for common
> implementations between graphdrivers, rather than specialized ones.
> This is less of a problem for performance since direct access to
> changesets is provided in the interface.
> 
> The Snapshotter provides an API for allocating, snapshotting and
> mounting abstract, layer-based filesystems. The model works by
> building up sets of directories with parent-child relationships, known
> as Snapshots.

### 整体架构和Model

![image.png-119.7kB][1]
![image.png-104.8kB][2]

> Snapshots are best understood by their lifecycle. Active snapshots are
> always created with Prepare or View from a Committed snapshot
> (including the empty snapshot). Committed snapshots are always created
> with Commit from an Active snapshot. Active snapshots never become
> committed snapshots and vice versa. All snapshots may be removed.
> 
> After mounting an Active snapshot, changes can be made to the
> snapshot. The act of committing creates a Committed snapshot. The
> committed snapshot will inherit the parent of the active snapshot. The
> committed snapshot can then be used as a parent. Active snapshots can
> never be used as a parent.
> 
> In this diagram, you can see that the active snapshot a is created by
> calling Prepare with the committed snapshot P0. After modification, a
> becomes a' and a committed snapshot P1 is created by calling Commit.
> a' can be further modified as a'' and a second committed snapshot can
> be created as P2 by calling Commit again. Note here that P2's parent
> is P0 and not P1.

要搞清楚这个p2的parent为什么是p0，而不是p1? 
--- 本质上a''还是从a做改动来的，而p2是会继承a''的parent的，也就是a的来源，p0 snapshot。

> Types of container filesystems In the container world we use two types
> of filesystems: overlays and snapshotting filesystems. AUFS and
> OverlayFS are overlay filesystems which have multiple directories with
> file diffs for each “layer” in an image. Snapshotting filesystems
> include devicemapper, btrfs, and ZFS which handle file diffs at the
> block level. Overlays usually work on common filesystem types such as
> EXT4 and XFS whereas snapshotting filesystems only run on volumes
> formatted for them.

快照类型的文件系统需要底层快设备提前format成对应的特定格式，不能直接运行在common filsystem上比如ext4等

目前支持的存储引擎有哪些？

* native 对应着vfs?对每个image layer直接无脑拷贝，不存在COW的语义
* lcow http://dockone.io/article/3299, https://github.com/moby/moby/pull/34859
* overlay
* aufs
* btrfs
* zfs

### 基本流程

#### Importing a Layer
To import a layer, we simply have the Snapshotter provide a list of mounts to be applied such that our destination will capture a changeset. We start out by getting a path to the layer tar file and creating a temp location to unpack it to:

    layerPath, tmpDir := getLayerPath(), mkTmpDir() // just a path to layer tar file.

We start by using a Snapshotter to Prepare a new snapshot transaction, using a key and descending from the empty parent "":

    mounts, err := snapshotter.Prepare(key, "")
    if err != nil { ... }

We get back a list of mounts from Snapshotter.Prepare, with the key identifying the active snapshot. Mount this to the temporary location with the following:

    if err := mount.All(mounts, tmpDir); err != nil { ... }

Once the mounts are performed, our temporary location is ready to capture a diff. In practice, this works similar to a filesystem transaction. The next step is to unpack the layer. We have a special function unpackLayer that applies the contents of the layer to target location and calculates the DiffID of the unpacked layer (this is a requirement for docker implementation):

    layer, err := os.Open(layerPath)
    if err != nil { ... }
    digest, err := unpackLayer(tmpLocation, layer) // unpack into layer location
    if err != nil { ... }

When the above completes, we should have a filesystem the represents the contents of the layer. Careful implementations should verify that digest matches the expected DiffID. When completed, we unmount the mounts:

    unmount(mounts) // optional, for now

Now that we've verified and unpacked our layer, we commit the active snapshot to a name. For this example, we are just going to use the layer digest, but in practice, this will probably be the ChainID:

    if err := snapshotter.Commit(digest.String(), key); err != nil { ... }

Now, we have a layer in the Snapshotter that can be accessed with the digest provided during commit. Once you have committed the snapshot, the active snapshot can be removed with the following:

    snapshotter.Remove(key)
    
从上边的描述我们可以看出，上一节unpack中的提到的step1-step3中只有step1 prepare以及step3 commit属于snapshotter的工作范畴，而step2的apply实际上调用的diff服务实现的接口。

#### Importing the Next Layer
Making a layer depend on the above is identical to the process described above except that the parent is provided as parent when calling Snapshotter.Prepare, assuming a clean tmpLocation:

    mounts, err := snapshotter.Prepare(tmpLocation, parentDigest)

We then mount, apply and commit, as we did above. The new snapshot will be based on the content of the previous one.

#### Running a Container
To run a container, we simply provide Snapshotter.Prepare the committed image snapshot as the parent. After mounting, the prepared path can be used directly as the container's filesystem:

    mounts, err := snapshotter.Prepare(containerKey, imageRootFSChainID)

The returned mounts can then be passed directly to the container runtime. If one would like to create a new image from the filesystem, Snapshotter.Commit is called:

    if err := snapshotter.Commit(newImageSnapshot, containerKey); err != nil { ... }

Alternatively, for most container runs, Snapshotter.Remove will be called to signal the Snapshotter to abandon the changes.

#### ctr snapshot命令

    COMMANDS:
         commit            commit an active snapshot into the provided name
         info              get info about a snapshot
         list, ls          list snapshots
         mounts, m, mount  mount gets mount commands for the snapshots
         prepare           prepare a snapshot from a committed snapshot
         remove, rm        remove snapshots
         label             add labels to content
         tree              display tree view of snapshot branches
         unpack            unpack applies layers from a manifest to a snapshot
         usage             usage snapshots
         view              create a read-only snapshot from a committed snapshot

涵盖了上文中介绍过的operation.

### 参考
https://github.com/containerd/containerd/blob/master/design/snapshots.md
https://integratedcode.us/2016/08/30/storage-drivers-in-docker-a-deep-dive/
https://www.cnblogs.com/breezey/p/9589288.html

  [1]: http://static.zybuluo.com/myecho/yo7ugmvzsg9fauj7b5nqv7x6/image.png
  [2]: http://static.zybuluo.com/myecho/qkfamwyg8x3f3vb2ylctkyqg/image.png

