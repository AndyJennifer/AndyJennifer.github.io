---
title: Android App 安装过程
tags:
  - null
categories:
  - null
---

### 前言

```java
    void installStage(String packageName, File stagedDir,
            IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
            String installerPackageName, int installerUid, UserHandle user,
            PackageParser.SigningDetails signingDetails) {
        if (DEBUG_INSTANT) {
            if ((sessionParams.installFlags & PackageManager.INSTALL_INSTANT_APP) != 0) {
                Slog.d(TAG, "Ephemeral install of " + packageName);
            }
        }
        final VerificationInfo verificationInfo = new VerificationInfo(
                sessionParams.originatingUri, sessionParams.referrerUri,
                sessionParams.originatingUid, installerUid);

        final OriginInfo origin = OriginInfo.fromStagedFile(stagedDir);
        //👇第一处
        final Message msg = mHandler.obtainMessage(INIT_COPY);
        final int installReason = fixUpInstallReason(installerPackageName, installerUid,
                sessionParams.installReason);
        //👇第二处
        final InstallParams params = new InstallParams(origin, null, observer,
                sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
                verificationInfo, user, sessionParams.abiOverride,
                sessionParams.grantedRuntimePermissions, signingDetails, installReason);
        params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
        msg.obj = params;

        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
                System.identityHashCode(msg.obj));
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(msg.obj));

        mHandler.sendMessage(msg);
    }

```

解释说明：

- 图中 1 处创建了类型为 `INIT_COPY` 的 Message。
- 图中 2 处创建 `InstallParams`，并传入安装包的相关数据。



### 最后

站在巨人的肩膀上，才能看的更远~
