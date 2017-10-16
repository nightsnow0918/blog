---
title: 將本機的程式放到 GCE VM Instance 上面
tags: [GCE, GCP]
---

最近在解的一個 issue 要處理 datastore 上的舊資料，需要長時間跑同一個 script (以天計)。
原本是用本機來跑，但一來本機資源畢竟比較吃緊，二來我也不可能 24 小時開著電腦去跑它，而公司沒有架設工作站，
大部份的服務都是建立在 GCP 上。所以後來就決定乾脆把 script 上傳到 Google compute engineer 上面跑，
反正 GCE 的計費和 CPU 使用率無關，開了卻不跑它反而浪費。於是就研究了一下怎麼把檔案上傳到 GCE，
還有怎麼從本機 mount 上 GCE 來下 command (就不用跑 cloud shell 了)

## 上傳檔案到 GCE VM instance
首先要先產生一組 SSH key，產生方式可以參考這個[連結](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys#createsshkeys)
- 產生的 public key 要貼到 GCP > Compute Engine > Metadata 底下
- 產生 key 流程期間會要你輸入一個 passphrase，必須記下，因為之後每次下 `scp` 都會要你輸入。

可以用 Linux 的 `scp`，完整流程可以參考這個[連結](https://cloud.google.com/compute/docs/instances/transfer-files#scp)
- Command:

    ```
    scp -i [SSH_KEY_FILE] [LOCAL_FILE_PATH] [USERNAME]@[IP_ADDRESS]:~
    ```

    以上就是將 `LOCAL_FILE_PATH` 上傳到對應的 VM instance (IP: `IP_ADDRESS`)的家目錄底下。其中 `SSH_KEY_FILE` 就是由第一步產生出來的 SSH key file，一般會放到 `~/.ssh/` 底下。

    若要抓整個資料夾的檔案則可以在來源檔案前面加上 `-rp`:

    ```
    scp -i ~/.ssh/my-ssh-key -rp [LOCAL_FILE_DIRECTORY] [USERNAME]@[IP_ADDRESS]:~
    ```

    若要從 VM 抓檔案到本機就把來源和目的反過來即可。
    - 每次下這個 command 都要輸入 passphrase。

接下來有兩種方式可以操作 GCE 上面的檔案
  
## 在 GCE instance 上面執行程式
- 要注意的是，如果程式有用到 GCP client API，則必須暫時把 Service account 改為自己的帳號 (才能 access 其它 gcloud service)

  ```
  gcloud auth application-default login
  ```
  
  選擇平常登入 GCP console 使用的帳號，再執行程式，才不會出現 403 permission error。不過這種作法會在 GCP > Compute Engine > Metadata 存入一組新的 public key (每次做就會有新的，會列出 Expire time)，為了避免產生過多多餘的 public key，也許改用 3. 的方法會比較好。

## 由本機 mount 上 GCE instance
有兩個方式可以做到:

### 更改本機的 gcloud config 再 SSH

  當然本機要先安裝好 gcloud 套件。安裝完成後在 command line 執行:

  ```
  gcloud compute ssh [INSTANCE_NAME]
  ```
  
  注意這邊是 `INSTANCE_NAME`，所以要先確定你的 gcloud config 有設成該 instance 對應到的值，像
  project 和 region/zone，才能連。設定 config 的命令如下:

  ```
  gcloud config set project [PROJECT_NAME]
  gcloud config set compute/region [REGION]
  gcloud config set compute/zone [ZONE]
  ```

  因為 REGION 和 ZONE 是在 `compute` session 底下所以要加上 `compute/` 在前面 (`project` 則是在 `core` 底下所以不用加)

### 在 Google Cloud Console 查看指令
  如果不想改這麼多 local gcloud 的設定，也可以直接到 GCE 的 SSH 選項查看連線的 command，如下圖:

    ![](https://i.imgur.com/J9lrayf.png)

  選擇「查看 gcloud 指令」，會跳出新對話框顯示連線至這台 VM 需要下的指令，複製貼上到本機的 console 就可以了。


不過 GCP 似乎會不定期把你的 thread 停掉，還是得隨時上去查看程式還有沒有在跑...
但至少就不用在自己的電腦上跑了。