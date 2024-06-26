---
title: Vue前端直连阿里云OSS
date: 2024/06/20 13:30
updated: 2024/06/20 13:30
tags: [前端开发]
categories: 开发
description: 强烈不推荐非专业安全人员尝试这种操作!
cover: https://i3.wp.com/wx2.sinaimg.cn/large/0072Vf1pgy1foxlnkwh9ij31hc0u0tqu.jpg
---

##  背景

有一个Web后台项目，之前的上传架构是用户前端上传->后端接口->后端调用oss.put把文件传到OSS中，非常合理安全的架构。但这样相当于后端服务器需要下载一次+上传一次，虽然OSS上传不收费，但如果用户上传了一个1G的文件，那么我的后端服务器会消耗2G流量，按照阿里云0.8元/G的流量价格，上传一个1G文件就是1.6元，这实在太贵了。


迫于没钱，我决定由前端直接连接OSS上传，不经过后端服务器，这样上传一个1G文件的成本是0

**警告: 强烈不推荐非专业安全人员尝试这种操作!**

**警告: 强烈不推荐非专业安全人员尝试这种操作!**

**警告: 强烈不推荐非专业安全人员尝试这种操作!**


##  实际方案

最标准的做法是： 后端新增一个需鉴权接口用于前端实时获取STS TOKEN，前端使用STS TOKEN直连OSS进行上传

我精简了一下，直接从后端接口中取出一个低权限的AKSK，再在前端初始化OSSClient。相比STS TOKEN方案，安全性有所下降但还能接受


这个方案是经过仔细考量的:

1. 这是一个后台系统，用户不登录成功的话无法使用任何前端页面的操作
2. 我的AKSK上传凭据是RAM用户，仅有OSS的管理权限
3. 获取AKSK的接口需要鉴权


```js
// UploadModal.vue
import { put, signatureUrl, getFileNameUUID } from '/@/utils/lib/oss'
import { reportUploadOSS } from '/@/api/user/xxxxx';

async function uploadApiByItem(item: FileItem) {
try {
    item.status = UploadResultStatus.UPLOADING;
    let obj_file_id = getFileNameUUID()
    let objName = obj_file_id + '_' + item.file.name
    await put(`${objName}`, item.file);
    const res2 = await signatureUrl(`${objName}`);
    await reportUploadOSS({
        file_name: item.file.name,
        file_id: obj_file_id,
        file_url: res2,
        file_type: item.file.file_type
    });

    item.status = UploadResultStatus.SUCCESS;
    item.responseData = res2;
    return {
        success: true,
        error: null
    };
} catch (e) {
    console.log(e);
    item.status = UploadResultStatus.ERROR;
    return {
        success: false,
        error: e,
    };
}
}
```


```js
// oss.js
import OSS from 'ali-oss'
import { GetOSSAKSK } from '/@/api/user/xxxxxx';
let get_aksk_res = await GetOSSAKSK()
let client = new OSS({
  region: get_aksk_res.region,
  secure: true,
  accessKeyId: get_aksk_res.accessKeyId,
  accessKeySecret: get_aksk_res.accessKeySecret,
  bucket: get_aksk_res.bucket
})

export const put = async (ObjName, fileUrl) => {  
try {    
    let result = await client.put(`${ObjName}`, fileUrl)    
    return result  
} catch (e) {    
    console.log(e)  
}
}

export const signatureUrl= async (ObjName) => {    
try {        
    let result = await client.signatureUrl(`${ObjName}`,{expires: 999999999})    
    return result  
} catch (e) {    
    console.log(e)  
}
}

export const getFileNameUUID = () => {  
    function rx() {    
      return (((1 + Math.random()) * 0x10000) | 0).toString(16).substring(1)  
    }  
    return `${+new Date()}_${rx()}${rx()}`
}
```



```py
@user_bp.route('/api/user/get_oss_secret', methods=['GET'])
@jwt_required()
def get_oss_secret():
    """获取AKSK信息"""
    return jsonify({'code': 200, 'msg': '', 'data': {"region": ALI_OSS_REGION, "accessKeyId": ALI_OSS_ACCESS_KEY,
                                                     "accessKeySecret": ALI_OSS_SECRET_KEY,
                                                     "bucket": ALI_OSS_BUCKET_NAME}})

```

## 其它错误方案

1. 将AKSK直接写在oss.js中，省去后端新加接口的成本 -> 所有攻击者可以伪造登录请求的返回值，进而加载到oss.js
2. 后端新接口返回的AKSK权限过大 -> 通过鉴权的攻击者可以直接接管服务器