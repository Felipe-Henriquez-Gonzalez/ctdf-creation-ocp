# CTFd 3.7.7 en OpenShift 4.21

## Requisitos
- `oc login` contra el clúster, con permisos para crear recursos en el namespace.
- Python `kubernetes`, `PyYAML`, Ansible y colección `kubernetes.core`.
- StorageClass `ocs-external-storagecluster-ceph-rbd` (ajustable).
- Los permisos de SCC y el `fsGroup` del PVC deben ser compatibles con el clúster.

```bash
python3 -m pip install kubernetes PyYAML
ansible-galaxy collection install -r requirements.yml
oc login https://API_DEL_CLUSTER:6443
export CTFD_DB_PASSWORD='UNA_CLAVE_LARGA_Y_SEGURA'
export CTFD_SECRET_KEY="$(openssl rand -hex 32)"
ansible-playbook deploy.yml
```

**Importante:** mantenga las mismas variables de secreto en ejecuciones posteriores. Cambiar `CTFD_DB_PASSWORD` en una base de datos ya inicializada NO cambia la contraseña del usuario dentro de MariaDB. Guarde los secretos en Ansible Vault o un gestor de secretos.

## Verificación
```bash
oc get pods,deploy,svc,pvc,route -n ctfd
oc exec -n ctfd deploy/redis -- redis-cli CONFIG GET save
oc exec -n ctfd deploy/redis -- redis-cli SET healthcheck ok
oc exec -n ctfd deploy/ctfd -- sh -c 'touch /var/uploads/.write-test && rm /var/uploads/.write-test'
oc logs -n ctfd deploy/ctfd --tail=100
```

## Advertencias y decisiones
- Redis sin RDB ni AOF: se pierden caché y sesiones al reiniciar.
- `fsGroup` no garantiza escritura en todos los CSI ni repara permisos internos ya existentes. Si falla la prueba de escritura, inspeccione `id`, `ls -ldn /var/uploads`, SCC y política CSI; no use `chmod 777` ni privilegios root como solución automática.
- `db_fs_group=999` y la imagen MariaDB elegida deben validarse en la SCC y almacenamiento reales. La imagen Docker oficial puede requerir ajustes de UID/SCC o imagen compatible con OpenShift. El despliegue de MariaDB en esta receta es una base propuesta, no una reproducción verificada de la imagen original.
- No se crean RoleBindings SCC `anyuid` automáticamente; si la imagen lo requiere, un administrador debe autorizar explícitamente la ServiceAccount correspondiente, siguiendo el principio de mínimo privilegio.
- `k8s state: present` no borra PVC. Sin embargo, actualizar definiciones existentes puede fallar si difieren campos inmutables. Haga respaldo antes de cambios a una instalación en uso.
- La importación de un backup CTFd puede sobrescribir datos. Haga un respaldo previo.
