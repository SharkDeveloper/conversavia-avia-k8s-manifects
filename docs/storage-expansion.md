# Настройка расширения томов local-path

## Проблема
StorageClass `local-path` по умолчанию не поддерживает изменение размера PVC "на лету" (allowVolumeExpansion: false).

## Решение

### Вариант 1: Включение allowVolumeExpansion в существующем StorageClass

Выполните команду для проверки текущего состояния:
```bash
kubectl get storageclass local-path -o yaml
```

Если `allowVolumeExpansion: false`, выполните:
```bash
kubectl patch storageclass local-path -p '{"allowVolumeExpansion": true}'
```

### Вариант 2: Настройка local-path-provisioner (для K3s)

Для K3s необходимо отредактировать ConfigMap провиженера:

```bash
kubectl -n kube-system edit configmap local-path-config
```

Затем обновите StorageClass:
```bash
kubectl patch storageclass local-path -p '{"allowVolumeExpansion": true}'
```

## Проверка

После настройки проверьте:
```bash
kubectl get storageclass local-path -o jsonpath='{.allowVolumeExpansion}'
```

Должно вернуть: `true`

## Использование

Теперь можно изменить размер PVC:
```bash
kubectl patch pvc paperless-ngx-data -n paperless-ngx -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

## Важные замечания

1. **Расширение возможно только в большую сторону** - уменьшить том нельзя
2. **Файловая система должна поддерживать online-resize** - ext4 и xfs поддерживают
3. **Под может потребовать перезапуска** - для применения нового размера
4. **Резервное копирование** - всегда делайте бэкап перед изменением размеров
