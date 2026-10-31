```
ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow/ruby_api $ gcloud auth login
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=32555940559.apps.googleusercontent.com&redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&scope=openid+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fsqlservice.login+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&state=wbr4gFDB9CeIEGVWIUjzZnRwQ7HCWz&access_type=offline&code_challenge=YtlTTUMaFcd-x1IQR3p43vsFZ_1D_ekULfkQmPgwkVw&code_challenge_method=S256


You are now logged in as [anantarala@gmail.com].
Your current project is [client-portal-484417].  You can change this setting by running:
  $ gcloud config set project PROJECT_ID
ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow/ruby_api $ printf '%s' 'YOUR_SECRET' | gcloud secrets create leadflow-api-secret --data-file=-
ERROR: (gcloud.secrets.create) [anantarala@gmail.com] does not have permission to access projects instance [client-portal-484417] (or it may not exist): Permission 'secretmanager.secrets.create' denied on resource (or it may not exist). Remediate access with this Troubleshooter URL or share it with your administrator - https://console.cloud.google.com/iam-admin/troubleshooter/summary;errorId=CiQwMWExOTI2YS0xMGE2LTc1NjItODcwNi1mYjcxMDk4NzZiZGMSAA%3D%3D . This command is authenticated as anantarala@gmail.com which is the active account specified by the [core/account] property.
- '@type': type.googleapis.com/google.rpc.ErrorInfo
  domain: iam.googleapis.com
  metadata:
    error_info_id: CiQwMWExOTI2YS0xMGE2LTc1NjItODcwNi1mYjcxMDk4NzZiZGMSAA==
    permission: secretmanager.secrets.create
    troubleshooter_url: https://console.cloud.google.com/iam-admin/troubleshooter/summary;errorId=CiQwMWExOTI2YS0xMGE2LTc1NjItODcwNi1mYjcxMDk4NzZiZGMSAA%3D%3D
  reason: IAM_PERMISSION_DENIED
ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow/ruby_api $ 

```
