---
title: TiDB-run!
date: 2018-03-25 14:01:47
tags:
- IT
- DB
categories:
- IT
---

[TiDB Source Code](https://github.com/pingcap/tidb) 

### Compute
        compute

### Storage

        storage

### Scheduling
        
        scheduling

####
``` golang
    func newStoreWithRetry(path string, maxRetries int) (kv.Storage, error) {
        storeURL, err := url.Parse(path)
        if err != nil {
            return nil, err
        }
    
        name := strings.ToLower(storeURL.Scheme)
        d, ok := stores[name]
        if !ok {
            return nil, errors.Errorf("invalid uri format, storage %s is not registered", name)
        }
    
        var s kv.Storage
        err = util.RunWithRetry(maxRetries, util.RetryInterval, func() (bool, error) {
            logutil.BgLogger().Info("new store", zap.String("path", path))
            s, err = d.Open(path)
            return kv.IsTxnRetryableError(err), err
        })
    
        if err == nil {
            logutil.BgLogger().Info("new store with retry success")
        } else {
            logutil.BgLogger().Warn("new store with retry failed", zap.Error(err))
        }
        return s, errors.Trace(err)
    }
```