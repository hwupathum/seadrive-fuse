libsearpc: A simple C language RPC framework (including both server side & client side)
autoconf:
automake:
libtool:

G Hash Tables
https://people.gnome.org/~ryanl/glib-docs/glib-Hash-Tables.html
------------------
G File Utilities
https://developer.gnome.org/glib/stable/glib-File-Utilities.html



==================
main {
    <!-- Create seafile-session object -->
    <!-- seafile_session_prepare (SeafileSession) -->
    <!-- seafile_session_start (SeafileSession) -->
    <!-- start_searpc_server -->
}
------------------

SeafileSession
------------------
SeafBlockManager    *block_mgr;
SeafFSManager       *fs_mgr;
SeafCommitManager   *commit_mgr;
SeafBranchManager   *branch_mgr;
SeafRepoManager     *repo_mgr; [x]
SeafSyncManager     *sync_mgr; [x]

SeafJobManager     *job_mgr;

HttpTxManager       *http_tx_mgr;
SeafFilelockManager *filelock_mgr;
FileCacheMgr        *file_cache_mgr; [x]
JournalManager *journal_mgr;
MqMgr               *mq_mgr;

session_start {
    <!-- http_tx_manager_start (HttpTxManager) -->
    <!-- seaf_sync_manager_start (SeafSyncManager) -->
    <!-- seaf_repo_manager_start (SeafRepoManager) -->
    <!-- seaf_filelock_manager_start (SeafFilelockManager) -->
    <!-- file_cache_mgr_start (FileCacheMgr) -->
}
-------------------


SeafSyncManager
-------------------
init {}
start {
    check_space_usage_thread {
        <!-- use NULL -->
        <!-- check current space usage each 60s -->
        <!-- http_tx_manager_api_get_space_usage (TxManager) -->
        <!-- seaf_repo_manager_set_account_space (RepoManager) -->
    }
    lock_file_worker thread {
        <!-- use mgr->priv->lock_file_job_queue -->
        <!-- check the queue and lock/unlock files accordingly -->
    }
    cache_file_task_worker thread {
        <!-- use mgr->priv->cache_file_task_queue -->
        <!-- seaf_repo_manager_get_repo (RepoManager) -->
        <!-- check connecticity -->
        <!-- repo_tree_traverse (RepoTree.h) and callback function schedule_cache_file -->
    }
}
schedule_cache_file {
    <!-- if file file_cache_mgr_cache_file  (FileCacheMgr)-->
    <!-- else file_cache_mgr_mkdir  (FileCacheMgr)-->
}
seaf_sync_manager_delete_active_path {

}
-------------------

SeafRepoManager
-------------------
/*
 * Repo object life cycle management:
 *
 * Each repo object has a refcnt. A repo object is inserted into an internal
 * hash table in repo manager when it's created or loaded from db on startup.
 * The initial refcnt is 1, for the reference by the internal hash table.
 *
 * A repo object can only be obtained from outside by calling
 * seaf_repo_manager_get_repo(), which increase refcnt by 1.
 * Once the caller is done with the object, it must call seaf_repo_unref() to
 * decrease refcnt.
 *
 * When a repo needs to be deleted, call seaf_repo_manager_mark_repo_deleted().
 * The function sets the "delete_pending" flag of the repo object.
 * Repo manager won't return the repo to callers once it's marked as deleted.
 * Repo manager regularly checks every repo in the hash table.
 * If a repo is marked as delete pending and refcnt is 1, it removes the repo
 * on disk. After the repo is removed on disk, repo manager removes the repo
 * object from internal hash table.
 *
 * Sync manager must make sure a repo won't be downloaded when it's marked as
 * delete pending. Otherwise repo manager may remove the downloaded metadata
 * as it cleans up the old repo on disk.
 */

init {
    <!-- Load all the repos into memory on the client side. -->
    <!-- Load folder permissions from db. -->
}
start {
    <!-- cleanup deleted stores by types commits, fs, blocks -->
    <!-- Delete repoos which are marked as delete pending and only referenced by the internal hash table. -->
}
seaf_repo_manager_set_account_space {
    <!-- sqlite replace into AccountSpace -->
}
seaf_repo_manager_get_repo {
    <!-- lock repo cache -->
    <!-- get the reop from RepoManager hash table -->
    <!-- release lock -->
}
-------------------

RepoTree
-------------------
<!-- RepoTree is a in-memory representation of the directory hierarchy of a repo. -->
repo_tree_traverse {
    <!-- do a tree traversal from root -->
    <!-- check if directory or not and run callback function -->
    <!-- recursive traverse -->
}
-------------------

FileCacheMgr
-------------------
<!-- MAX_THREADS 3 -->
<!-- DEFAULT_CLEAN_CACHE_INTERVAL 600 // 10 minutes -->
<!-- DEFAULT_CACHE_SIZE_LIMIT (10000000000LL) // 10GB -->
<!-- REMOVE_DELETED_CACHE_INTERVAL 30 // 30 seconds -->
<!-- BEFORE_NOTIFY_DOWNLOAD_START 5 -->
new {
    fetch_file_worker thread pool {
        <!-- seaf_sync_manager_get_server_info (SeafSyncManager) -->
        <!-- seaf_repo_manager_get_repo (SeafRepoManager) -->
        <!-- check if repo encrypted -->
        <!-- set repo tree path in repo -->
        <!-- seaf_fs_manager_get_seafile (SeafFSManager) -->
        <!-- get cache block map from server. create if not availabe -->
        <!-- download blocks from the repo -->
    }
<!-- create file cache directory -->
}
init {
    <!-- get clean interval and cache size from config -->
}
start {
    clean_cache_worker thread { 
        <!-- Delay 1 minute so that it doesn't conflict with repo tree loading -->
        <!-- ==================================== IMPORTANT ==================================== -->
        <!-- do_clean_cache after clean cache interval time -->
            <!-- Clean until the total cache size reduce to 70% of limit. -->
                <!--  Removing cached_file from the hash table will decrease n_open by 1.
                When all open handle to this file is closed, the cached file structure
                will be freed.
                If later a file with the same name is created and opened, a new cache file
                structure will be allocated.
                -->
                <!-- The cached file on disk cannot be found after deletion. Access to the
                old file path will fail since read/write will check extended attrs of
                the ondisk cached file. This is incompatible with POSIX semantics but
                is simpler to implement. We don't have to track changes to file paths
                in this way.
                -->
            <!-- seaf_sync_manager_delete_active_path (SyncManager) -->
    }
    remove_deleted_cache_worker thread {
        <!-- do_remove_deleted_cache after REMOVE_DELETED_CACHE_INTERVAL time -->
            <!-- reaf_rm_recursive (Utils) - remove directories and files inside recursively from cache -->
    }
    check_download_file_time_worker thread {
        <!-- use CachedFileHandle -->
        <!-- check when to download a file? -->
        <!-- send_file_download_notification (file-download.start) to messageQueue (RpcService) -->
    }
}
start_cache_task {
    <!-- Only one fetch task can be running for each file.  -->
    <!-- If the cached file is up-to-date, don't need to fetch again.  -->
    <!-- If cached file is in "FETCHING" status, its download was interrupted by restart. -->
    <!-- Only non-zero size file needs to be fetched from server. -->
    <!-- add thread to check_download_file_time_worker pool -->
}
start_cache_task_before_read {
    <!-- On Windows, read operation may be issued after close is called.
    So sometimes when we read, the cache task may have been canceled.
    To ensure the file will be cached, we try to start cache task
    for every read operation. Since we make sure there can be only one
    cache task running for each file, it won't cause problems.
    -->
    <!-- start_cache_task -->
}
file_cache_mgr_cache_file {
    <!-- use SeafRepo -->
    <!-- check if file already cached -->
    <!-- add CachedFile to FileCacheMgr hash table -->
    <!-- start_cache_task -->
}
file_cache_mgr_mkdir {
    <!-- Create a directory if it doesn't already exist. Create intermediate parent directories as needed, too. -->
}
file_cache_mgr_rmdir {
    <!-- remove directory -->
}
file_cache_mgr_read_by_path {
    <!-- start_cache_task_before_read -->
    <!-- wait until file is cached -->
    <!-- read the file -->
}
file_cache_mgr_read {

}
file_cache_mgr_write {

}
file_cache_mgr_write_by_path {
    
}
-------------------

SeafFSManager
-------------------
<!-- CDC_AVERAGE_BLOCK_SIZE (1 << 23) /* 8MB */ -->
<!-- CDC_MIN_BLOCK_SIZE (6 * (1 << 20)) /* 6MB */ -->
<!-- CDC_MAX_BLOCK_SIZE (10 * (1 << 20)) /* 10MB */ -->

-------------------
