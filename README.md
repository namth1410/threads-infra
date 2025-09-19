# threads-infra
kubectl edit application ingress-nginx -n argocd => xóa ingerss

argo pass: 
```
╰─ kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo                                                                 ─╯

ibhLYtMlTn7mdpag

argocd login localhost:8080 --username admin --password ibhLYtMlTn7mdpag  
```


argo 8080
drone 8081
harbor 8082

### Minio
    MINIO_ROOT_USER: namth
    MINIO_ROOT_PASSWORD: 01664157092aA
### Postgres
    POSTGRES_USER: namth
    POSTGRES_PASSWORD: 01664157092aA
    POSTGRES_DB: threads_db