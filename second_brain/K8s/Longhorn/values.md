# Значения по умолчанию для Longhorn.
# Это YAML-файл конфигурации.
# Объявите переменные для передачи в шаблоны.
global:
  # -- Глобальное переопределение реестра образов контейнеров.
  imageRegistry: ""
  # -- Глобальное переопределение секретов для pull образов из реестра.
  imagePullSecrets: []
  # -- Tolerations для нод, на которых могут работать пользовательские компоненты Longhorn: Manager, UI, Driver Deployer.
  tolerations: []
  # -- Node selector для нод, на которых могут работать пользовательские компоненты Longhorn: Manager, UI, Driver Deployer.
  nodeSelector: {}
  cattle:
    # -- Системный реестр по умолчанию.
    systemDefaultRegistry: ""
    windowsCluster:
      # -- Настройка, позволяющая Longhorn работать в Windows кластере Rancher.
      enabled: false
      # -- Tolerations для Linux нод, на которых могут работать пользовательские компоненты Longhorn.
      tolerations:
        - key: "cattle.io/os"
          value: "linux"
          effect: "NoSchedule"
          operator: "Equal"
      # -- Node selector для Linux нод, на которых могут работать пользовательские компоненты Longhorn.
      nodeSelector:
        kubernetes.io/os: "linux"
      defaultSetting:
        # -- Tolerations для системных компонентов Longhorn.
        taintToleration: cattle.io/os=linux:NoSchedule
        # -- Node selector для системных компонентов Longhorn.
        systemManagedComponentsNodeSelector: kubernetes.io/os:linux
networkPolicies:
  # -- Настройка, позволяющая включить сетевые политики для контроля доступа к подам Longhorn.
  enabled: false
  # -- Дистрибутив, определяющий политику разрешения доступа для ingress. (Варианты: "k3s", "rke2", "rke1")
  type: "k3s"
image:
  longhorn:
    engine:
      # -- Реестр для образа Longhorn Engine.
      registry: ""
      # -- Репозиторий для образа Longhorn Engine.
      repository: longhornio/longhorn-engine
      # -- Тег для образа Longhorn Engine.
      tag: v1.10.1
    manager:
      # -- Реестр для образа Longhorn Manager.
      registry: ""
      # -- Репозиторий для образа Longhorn Manager.
      repository: longhornio/longhorn-manager
      # -- Тег для образа Longhorn Manager.
      tag: v1.10.1
    ui:
      # -- Реестр для образа Longhorn UI.
      registry: ""
      # -- Репозиторий для образа Longhorn UI.
      repository: longhornio/longhorn-ui
      # -- Тег для образа Longhorn UI.
      tag: v1.10.1
    instanceManager:
      # -- Реестр для образа Longhorn Instance Manager.
      registry: ""
      # -- Репозиторий для образа Longhorn Instance Manager.
      repository: longhornio/longhorn-instance-manager
      # -- Тег для образа Longhorn Instance Manager.
      tag: v1.10.1
    shareManager:
      # -- Реестр для образа Longhorn Share Manager.
      registry: ""
      # -- Репозиторий для образа Longhorn Share Manager.
      repository: longhornio/longhorn-share-manager
      # -- Тег для образа Longhorn Share Manager.
      tag: v1.10.1
    backingImageManager:
      # -- Реестр для образа Backing Image Manager. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа Backing Image Manager. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/backing-image-manager
      # -- Тег для образа Backing Image Manager. Если не указано, Longhorn использует значение по умолчанию.
      tag: v1.10.1
    supportBundleKit:
      # -- Реестр для образа Longhorn Support Bundle Manager.
      registry: ""
      # -- Репозиторий для образа Longhorn Support Bundle Manager.
      repository: longhornio/support-bundle-kit
      # -- Тег для образа Longhorn Support Bundle Manager.
      tag: v0.0.71
  csi:
    attacher:
      # -- Реестр для образа CSI attacher. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа CSI attacher. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/csi-attacher
      # -- Тег для образа CSI attacher. Если не указано, Longhorn использует значение по умолчанию.
      tag: v4.10.0-20251030
    provisioner:
      # -- Реестр для образа CSI Provisioner. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа CSI Provisioner. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/csi-provisioner
      # -- Тег для образа CSI Provisioner. Если не указано, Longhorn использует значение по умолчанию.
      tag: v5.3.0-20251030
    nodeDriverRegistrar:
      # -- Реестр для образа CSI Node Driver Registrar. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа CSI Node Driver Registrar. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/csi-node-driver-registrar
      # -- Тег для образа CSI Node Driver Registrar. Если не указано, Longhorn использует значение по умолчанию.
      tag: v2.15.0-20251030
    resizer:
      # -- Реестр для образа CSI Resizer. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа CSI Resizer. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/csi-resizer
      # -- Тег для образа CSI Resizer. Если не указано, Longhorn использует значение по умолчанию.
      tag: v1.14.0-20251030
    snapshotter:
      # -- Реестр для образа CSI Snapshotter. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа CSI Snapshotter. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/csi-snapshotter
      # -- Тег для образа CSI Snapshotter. Если не указано, Longhorn использует значение по умолчанию.
      tag: v8.4.0-20251030
    livenessProbe:
      # -- Реестр для образа CSI liveness probe. Если не указано, Longhorn использует значение по умолчанию.
      registry: ""
      # -- Репозиторий для образа CSI liveness probe. Если не указано, Longhorn использует значение по умолчанию.
      repository: longhornio/livenessprobe
      # -- Тег для образа CSI liveness probe. Если не указано, Longhorn использует значение по умолчанию.
      tag: v2.17.0-20251030
  openshift:
    oauthProxy:
      # -- Реестр для образа OAuth Proxy. Укажите upstream образ (например, "quay.io/openshift/origin-oauth-proxy"). Применяется только для OpenShift.
      registry: ""
      # -- Репозиторий для образа OAuth Proxy. Укажите upstream образ (например, "quay.io/openshift/origin-oauth-proxy"). Применяется только для OpenShift.
      repository: ""
      # -- Тег для образа OAuth Proxy. Укажите версию OCP/OKD 4.1 или выше (включая 4.18, доступную на quay.io/openshift/origin-oauth-proxy:4.18). Применяется только для OpenShift.
      tag: ""
  # -- Политика загрузки образов для всех пользовательских компонентов Longhorn: Manager, Driver, UI.
  pullPolicy: IfNotPresent
service:
  ui:
    # -- Тип сервиса для Longhorn UI. (Варианты: "ClusterIP", "NodePort", "LoadBalancer", "Rancher-Proxy")
    type: ClusterIP
    # -- NodePort порт для Longhorn UI. Если не указано, Longhorn выбирает свободный порт между 30000 и 32767.
    nodePort: null
    # -- Аннотации для сервиса Longhorn UI.
    annotations: {}
    ## Если хотите установить аннотации для сервиса Longhorn UI, удалите `{}` в строке выше
    ## и раскомментируйте этот блок примера
    #  annotation-key1: "annotation-value1"
    #  annotation-key2: "annotation-value2"
  manager:
    # -- Тип сервиса для Longhorn Manager.
    type: ClusterIP
    # -- NodePort порт для Longhorn Manager. Если не указано, Longhorn выбирает свободный порт между 30000 и 32767.
    nodePort: ""
persistence:
  # -- Настройка, позволяющая указать Longhorn StorageClass как класс по умолчанию.
  defaultClass: true
  # -- Тип файловой системы для Longhorn StorageClass по умолчанию.
  defaultFsType: ext4
  # -- Параметры mkfs для Longhorn StorageClass по умолчанию.
  defaultMkfsParams: ""
  # -- Количество реплик для Longhorn StorageClass по умолчанию.
  defaultClassReplicaCount: 2
  # -- Локальность данных для Longhorn StorageClass по умолчанию. (Варианты: "disabled", "best-effort")
  defaultDataLocality: disabled
  # -- Политика освобождения тома после удаления claim. (Варианты: "Retain" - сохранить, "Delete" - удалить)
  reclaimPolicy: Delete
  # -- VolumeBindingMode контролирует когда происходит привязка тома и динамическое создание. (Варианты: "Immediate", "WaitForFirstConsumer") (По умолчанию: "Immediate")
  volumeBindingMode: "Immediate"
  # -- Настройка, позволяющая включить живую миграцию тома Longhorn с одной ноды на другую.
  migratable: false
  # -- Настройка, отключающая счётчик ревизий, что предотвращает отслеживание всех операций записи в том. При восстановлении тома Longhorn использует свойства файла volume-head-xxx.img (размер и время модификации) для выбора реплики для восстановления.
  disableRevisionCounter: "true"
  # -- Установить NFS опции монтирования для Longhorn StorageClass для RWX томов
  nfsOptions: ""
  recurringJobSelector:
    # -- Настройка, позволяющая включить селектор периодических задач для Longhorn StorageClass.
    enable: false
    # -- Селектор периодических задач для Longhorn StorageClass. Убедитесь, что кавычки используются правильно при указании параметров задач. (Пример: `[{"name":"backup", "isGroup":true}]`)
    jobList: []
  backingImage:
    # -- Настройка, позволяющая использовать backing image в Longhorn StorageClass.
    enable: false
    # -- Backing image для создания и восстановления томов в Longhorn StorageClass. Когда backing images недоступны, укажите тип источника данных и параметры, которые Longhorn может использовать для создания backing image.
    name: ~
    # -- Тип источника данных backing image в Longhorn StorageClass.
    # Если backing image существует в кластере, Longhorn использует эту настройку для проверки образа.
    # Если backing image не существует, Longhorn создаёт его используя указанный тип источника данных.
    dataSourceType: ~
    # -- Параметры источника данных backing image в Longhorn StorageClass.
    # Можно указать JSON строку map. (Пример: `'{\"url\":\"https://backing-image-example.s3-region.amazonaws.com/test-backing-image\"}'`)
    dataSourceParameters: ~
    # -- Ожидаемая SHA-512 контрольная сумма backing image в Longhorn StorageClass.
    expectedChecksum: ~
  defaultDiskSelector:
    # -- Настройка, позволяющая включить селектор дисков для Longhorn StorageClass по умолчанию.
    enable: false
    # -- Селектор дисков для Longhorn StorageClass по умолчанию. Longhorn использует только диски с указанными тегами для хранения данных томов. (Примеры: "nvme,sata")
    selector: ""
  defaultNodeSelector:
    # -- Настройка, позволяющая включить селектор нод для Longhorn StorageClass по умолчанию.
    enable: false
    # -- Селектор нод для Longhorn StorageClass по умолчанию. Longhorn использует только ноды с указанными тегами для хранения данных томов. (Примеры: "storage,fast")
    selector: ""
  # -- Настройка, позволяющая включить автоматическое удаление снапшотов при trim файловой системы для Longhorn StorageClass. (Варианты: "ignored", "enabled", "disabled")
  unmapMarkSnapChainRemoved: ignored
  # -- Настройка, позволяющая указать версию data engine для Longhorn StorageClass по умолчанию. (Варианты: "v1", "v2")
  dataEngine: v1
  # -- Настройка, позволяющая указать backup target для Longhorn StorageClass по умолчанию.
  backupTargetName: default
preUpgradeChecker:
  # -- Настройка, позволяющая Longhorn выполнять проверки перед обновлением. Отключите при установке Longhorn через Argo CD или другие GitOps решения.
  jobEnabled: true
  # -- Настройка, позволяющая Longhorn выполнять проверки версии после запуска подов DaemonSet Longhorn Manager. Отключение этой настройки также отключает `preUpgradeChecker.jobEnabled`. Рекомендуется держать включённым.
  upgradeVersionCheck: true
csi:
  # -- Корневая директория kubelet. Если не указано, Longhorn использует значение по умолчанию.
  kubeletRootDir: ~
  # -- Количество реплик CSI Attacher. Если не указано, Longhorn использует значение по умолчанию ("3").
  attacherReplicaCount: ~
  # -- Количество реплик CSI Provisioner. Если не указано, Longhorn использует значение по умолчанию ("3").
  provisionerReplicaCount: ~
  # -- Количество реплик CSI Resizer. Если не указано, Longhorn использует значение по умолчанию ("3").
  resizerReplicaCount: ~
  # -- Количество реплик CSI Snapshotter. Если не указано, Longhorn использует значение по умолчанию ("3").
  snapshotterReplicaCount: ~
defaultSettings:
  # -- Настройка, позволяющая Longhorn автоматически подключать том и создавать снапшоты или бэкапы при выполнении периодических задач.
  allowRecurringJobWhileVolumeDetached: ~
  # -- Настройка, позволяющая Longhorn автоматически создавать диск по умолчанию только на нодах с label "node.longhorn.io/create-default-disk=true" (если других дисков нет). Когда отключено, Longhorn создаёт диск по умолчанию на каждой ноде, добавленной в кластер.
  createDefaultDiskLabeledNodes: ~
  # -- Путь по умолчанию для хранения данных на хосте. Абсолютный путь к директории означает filesystem-диск для V1 Data Engine, путь к block device означает block-диск для V2 Data Engine. Значение по умолчанию: "/var/lib/longhorn/".
  defaultDataPath: /mnt/data
  # -- Локальность данных по умолчанию. Том Longhorn имеет локальность данных, если локальная реплика тома существует на той же ноде, что и под, использующий том.
  defaultDataLocality: ~
  # -- Настройка, позволяющая планирование на ноды со здоровыми репликами того же тома. Эта настройка отключена по умолчанию.
  replicaSoftAntiAffinity: ~
  # -- Настройка автоматической перебалансировки реплик при обнаружении доступной ноды.
  replicaAutoBalance: "disabled"
  # -- Процент хранилища, который может быть выделен относительно ёмкости диска. Значение по умолчанию: "100".
  storageOverProvisioningPercentage: ~
  # -- Минимальный процент доступной ёмкости диска. Когда минимальная доступная ёмкость превышает общую доступную ёмкость, диск становится недоступным для планирования пока не освободится место. Значение по умолчанию: "25".
  storageMinimalAvailablePercentage: ~
  # -- Процент места на диске, не выделяемый под диск по умолчанию на каждой новой ноде Longhorn.
  storageReservedPercentageForDefaultDisk: 0
  # -- Проверка обновлений, периодически проверяющая наличие новых версий Longhorn. При наличии новой версии появляется уведомление в UI Longhorn. Включено по умолчанию.
  upgradeChecker: ~
  # -- Upgrade Responder отправляет уведомление при появлении новой версии Longhorn. Значение по умолчанию: https://longhorn-upgrade-responder.rancher.io/v1/checkupgrade.
  upgradeResponderURL: ~
  # -- Количество реплик по умолчанию для томов, созданных через UI Longhorn. Для конфигурации Kubernetes измените поле `numberOfReplicas` в StorageClass. Значение по умолчанию: "{"v1":"3","v2":"3"}".
  defaultReplicaCount: 2
  # -- Имя статического StorageClass Longhorn по умолчанию. "storageClassName" присваивается PV и PVC, созданным для существующего тома Longhorn. "storageClassName" также может использоваться как label для привязки workload к существующему PV без создания объекта Kubernetes StorageClass. "storageClassName" должен быть существующим StorageClass. Значение по умолчанию: "longhorn-static".
  defaultLonghornStaticStorageClass: ~
  # -- Время хранения (минуты) неудачного ресурса бэкапа. При значении "0" автоматическое удаление отключено.
  failedBackupTTL: ~
  # -- Время (минуты), которое Longhorn отводит на выполнение бэкапа. Значение по умолчанию: "1".
  backupExecutionTimeout: ~
  # -- Настройка восстановления периодических задач из backup volume на backup target и создания периодических задач если их нет при восстановлении бэкапа.
  restoreVolumeRecurringJobs: ~
  # -- Максимальное количество успешных периодических задач бэкапа и снапшотов для хранения. При значении "0" история успешных задач не сохраняется.
  recurringSuccessfulJobsHistoryLimit: ~
  # -- Максимальное количество неудачных периодических задач бэкапа и снапшотов для хранения. При значении "0" история неудачных задач не сохраняется.
  recurringFailedJobsHistoryLimit: ~
  # -- Максимальное количество снапшотов или бэкапов для хранения.
  recurringJobMaxRetention: ~
  # -- Максимальное количество неудачных support bundles в кластере. При значении "0" Longhorn автоматически удаляет все неудачные support bundles.
  supportBundleFailedHistoryLimit: ~
  # -- Taint или toleration для системных компонентов Longhorn.
  # Укажите значения через точку с запятой в синтаксисе `kubectl taint` (Пример: key1=value1:effect; key2=value2:effect).
  taintToleration: ~
  # -- Node selector для системных компонентов Longhorn.
  systemManagedComponentsNodeSelector: ~
  # -- PriorityClass для системных компонентов Longhorn.
  # Эта настройка помогает предотвратить вытеснение компонентов Longhorn при нехватке ресурсов на ноде.
  # Обратите внимание, что это будет применено к пользовательским компонентам Longhorn по умолчанию, если значения priority class ещё не установлены, например `longhornManager.priorityClass`.
  priorityClass: &defaultPriorityClassNameRef "longhorn-critical"
  # -- Настройка, позволяющая Longhorn автоматически восстанавливать тома когда все реплики становятся неисправными (например, при обрыве сетевого соединения). Longhorn определяет какие реплики можно использовать и затем использует их для тома. Включено по умолчанию.
  autoSalvage: ~
  # -- Настройка, позволяющая Longhorn автоматически удалять под workload, управляемый контроллером (например, daemonset) при неожиданном отключении тома Longhorn (например, при обновлении Kubernetes). После удаления контроллер перезапускает под и Kubernetes обрабатывает повторное подключение и монтирование тома.
  autoDeletePodWhenVolumeDetachedUnexpectedly: ~
  # -- Чёрный список controller api/kind для настройки автоматического удаления пода при неожиданном отключении тома. Если под workload управляется контроллером, чей api/kind в этом списке, Longhorn не будет автоматически удалять под при неожиданном отключении тома. Можно указать несколько записей через точку с запятой. Например: `apps/StatefulSet;apps/DaemonSet`. Controller api/kind чувствителен к регистру и должен точно соответствовать api/kind в owner reference пода.
  blacklistForAutoDeletePodWhenVolumeDetachedUnexpectedly: ~
  # -- Настройка, запрещающая Longhorn Manager планировать реплики на cordoned Kubernetes нодах. Включено по умолчанию.
  disableSchedulingOnCordonedNode: ~
  # -- Настройка, позволяющая Longhorn планировать новые реплики тома на ноды в той же зоне, что и существующие здоровые реплики. Ноды без зоны считаются находящимися в зоне со здоровыми репликами. При определении зон Longhorn использует label "topology.kubernetes.io/zone=<Zone name of the node>" в объекте Kubernetes node.
  replicaZoneSoftAntiAffinity: ~
  # -- Настройка, позволяющая планирование на диски с существующими здоровыми репликами того же тома. Включено по умолчанию.
  replicaDiskSoftAntiAffinity: ~
  # -- Политика, определяющая действия Longhorn когда том застрял с подом StatefulSet или Deployment на упавшей ноде.
  nodeDownPodDeletionPolicy: ~
  # -- Политика, определяющая действия Longhorn при drain ноды с последней здоровой репликой тома.
  nodeDrainPolicy: ~
  # -- Настройка, позволяющая автоматическое отключение вручную подключённых томов при cordon ноды.
  detachManuallyAttachedVolumesWhenCordoned: ~
  # -- Количество секунд ожидания перед переиспользованием существующих данных на упавшей реплике вместо создания новой реплики деградированного тома.
  replicaReplenishmentWaitInterval: ~
  # -- Максимальное количество реплик, которые могут одновременно пересобираться на каждой ноде.
  concurrentReplicaRebuildPerNodeLimit: ~
  # -- Максимальное количество томов, которые могут одновременно восстанавливаться из бэкапа на каждой ноде. При значении "0" восстановление томов из бэкапа отключено.
  concurrentVolumeBackupRestorePerNodeLimit: ~
  # -- Настройка, отключающая счётчик ревизий и тем самым предотвращающая отслеживание всех операций записи в том. При восстановлении тома Longhorn использует свойства файла "volume-head-xxx.img" (размер и время модификации) для выбора реплики для восстановления. Эта настройка применяется только к томам, созданным через UI Longhorn.
  disableRevisionCounter: '{"v1":"true"}'
  # -- Политика загрузки образов для системных подов: Instance Manager, engine images, CSI Driver. Изменения политики применяются только после перезапуска системных подов.
  systemManagedPodsImagePullPolicy: ~
  # -- Настройка, позволяющая создавать и подключать том без всех запланированных реплик в момент создания.
  allowVolumeCreationWithDegradedAvailability: ~
  # -- Настройка, позволяющая Longhorn автоматически очищать системные снапшоты после завершения пересборки реплик.
  autoCleanupSystemGeneratedSnapshot: ~
  # -- Настройка, позволяющая Longhorn автоматически очищать снапшоты, созданные периодическими задачами бэкапа.
  autoCleanupRecurringJobBackupSnapshot: ~
  # -- Максимальное количество engines, которые могут одновременно обновляться на каждой ноде после обновления Longhorn Manager. При значении "0" Longhorn не обновляет автоматически engine томов до новой версии образа engine по умолчанию.
  concurrentAutomaticEngineUpgradePerNodeLimit: ~
  # -- Количество минут ожидания перед очисткой файла backing image когда ни одна реплика на диске не использует его.
  backingImageCleanupWaitInterval: ~
  # -- Количество секунд ожидания перед повторной загрузкой файла backing image когда статус всех файлов образа на дисках становится "failed" или "unknown".
  backingImageRecoveryWaitInterval: ~
  # -- Процент от общих выделяемых ресурсов CPU на каждой ноде для резервирования каждому поду instance manager. Значение по умолчанию: {"v1":"12","v2":"12"}.
  guaranteedInstanceManagerCPU: ~
  # -- Настройка, уведомляющая Longhorn что кластер использует Kubernetes Cluster Autoscaler.
  kubernetesClusterAutoscalerEnabled: ~
  # -- Включает автоматическое удаление Longhorn orphaned (осиротевших) ресурсов и связанных с ними данных или процессов (например, устаревших реплик). Orphaned ресурсы на упавших или unknown нодах не очищаются автоматически.
  # Нужно указать типы ресурсов для удаления через точку с запятой (например, `replica-data;instance`). Доступные типы: `replica-data`, `instance`.
  orphanResourceAutoDeletion: ~
  # -- Время ожидания (секунды) перед автоматическим удалением orphaned Custom Resource (CR) и связанных ресурсов.
  # Если пользователь вручную удаляет orphaned CR, удаление происходит немедленно без учёта этого периода.
  orphanResourceAutoDeletionGracePeriod: ~
  # -- Сеть хранения для внутрикластерного трафика. Если не указано, Longhorn использует сеть кластера Kubernetes.
  storageNetwork: ~
  # -- Флаг, предотвращающий случайное удаление Longhorn.
  deletingConfirmationFlag: ~
  # -- Таймаут между Longhorn Engine и репликами. Укажите значение от "8" до "30" секунд. Значение по умолчанию: "8".
  engineReplicaTimeout: ~
  # -- Настройка, позволяющая включить хеширование снапшотов и проверки целостности данных.
  snapshotDataIntegrity: ~
  # -- Настройка, отключающая хеширование снапшотов после их создания для минимизации влияния на производительность системы.
  snapshotDataIntegrityImmediateCheckAfterSnapshotCreation: ~
  # -- Настройка, определяющая когда Longhorn проверяет целостность данных в файлах снапшотов. Используйте формат Unix cron.
  snapshotDataIntegrityCronjob: ~
  # -- Настройка, позволяющая Longhorn автоматически помечать последний снапшот и его родительские файлы для удаления при trim файловой системы. Longhorn не удаляет снапшоты с несколькими дочерними файлами.
  removeSnapshotsDuringFilesystemTrim: ~
  # -- Настройка, позволяющая быструю пересборку реплик с использованием контрольных сумм файлов снапшотов. Перед включением установите snapshot-data-integrity в "enable" или "fast-check".
  fastReplicaRebuildEnabled: ~
  # -- Количество секунд ожидания HTTP клиентом ответа от сервера File Sync перед признанием соединения неудачным.
  replicaFileSyncHttpClientTimeout: ~
  # -- Количество секунд, которое Longhorn отводит на завершение операций пересборки реплик и клонирования снапшотов.
  longGRPCTimeOut: ~
  # -- Уровни логов, указывающие тип и серьёзность логов в Longhorn Manager. Значение по умолчанию: "Info". (Варианты: "Panic", "Fatal", "Error", "Warn", "Info", "Debug", "Trace")
  logLevel: ~
  # -- Директория на хосте где Longhorn хранит файлы логов для подов instance manager. Используется только для подов instance manager в V2 data engine.
  logPath: ~
  # -- Настройка, позволяющая указать метод сжатия бэкапов.
  backupCompressionMethod: ~
  # -- Максимальное количество рабочих потоков, которые могут одновременно работать для каждого бэкапа.
  backupConcurrentLimit: ~
  # -- Размер блока бэкапа по умолчанию (MiB) при создании нового тома. Поддерживаемые значения: 2 или 16.
  defaultBackupBlockSize: ~
  # -- Максимальное количество рабочих потоков, которые могут одновременно работать для каждой операции восстановления.
  restoreConcurrentLimit: ~
  # -- Настройка, позволяющая включить V1 Data Engine.
  v1DataEngine: ~
  # -- Настройка, позволяющая включить V2 Data Engine на основе Storage Performance Development Kit (SPDK). V2 Data Engine - экспериментальная функция, не использовать в продакшене.
  v2DataEngine: ~
  # -- Применяется только к V2 Data Engine. Включает hugepages для демона Storage Performance Development Kit (SPDK) target. Если отключено, используется legacy память. Размер выделения устанавливается через Data Engine Memory Size.
  dataEngineHugepageEnabled: ~
  # -- Применяется только к V2 Data Engine. Размер hugepage (MiB) для демона Storage Performance Development Kit (SPDK) target. Значение по умолчанию: "{"v2":"2048"}"
  dataEngineMemorySize: ~
  # -- Применяется только к V2 Data Engine. CPU ядра на которых работает демон Storage Performance Development Kit (SPDK) target. Демон разворачивается в каждом поде Instance Manager. Убедитесь что количество назначенных ядер не превышает гарантированные CPU Instance Manager для V2 Data Engine. Значение по умолчанию: "{"v2":"0x1"}".
  dataEngineCPUMask: ~
  # -- Лимит пропускной способности записи по умолчанию (мегабайт/с) для пересборки реплик тома при использовании V2 data engine (SPDK). При значении 0 лимит отсутствует. Отдельные тома могут переопределить эту настройку.
  replicaRebuildingBandwidthLimit: ~
  # -- В секундах. Таймаут для liveness probe пода instance manager. Значение по умолчанию: 10 секунд.
  instanceManagerPodLivenessProbeTimeout: ~
  # -- Настройка, позволяющая планировать тома с пустым node selector на любую ноду.
  allowEmptyNodeSelectorVolume: ~
  # -- Настройка, позволяющая планировать тома с пустым disk selector на любой диск.
  allowEmptyDiskSelectorVolume: ~
  # -- Настройка, позволяющая Longhorn периодически собирать анонимные данные об использовании для улучшения продукта. Longhorn отправляет данные на сервер [Upgrade Responder](https://github.com/longhorn/upgrade-responder), который является источником данных для Longhorn Public Metrics Dashboard (https://metrics.longhorn.io). Сервер Upgrade Responder не хранит данные для идентификации клиентов, включая IP адреса.
  allowCollectingLonghornUsageMetrics: ~
  # -- Настройка, временно предотвращающая все попытки очистки снапшотов томов.
  disableSnapshotPurge: ~
  # -- Максимальное количество снапшотов для тома. Значение должно быть от 2 до 250.
  snapshotMaxCount: ~
  # -- Применяется только к V2 Data Engine. Уровень логов для демона Storage Performance Development Kit (SPDK) target. Поддерживаемые значения: Error, Warning, Notice, Info, Debug. По умолчанию: Notice.
  dataEngineLogLevel: ~
  # -- Применяется только к V2 Data Engine. Флаги логов для демона Storage Performance Development Kit (SPDK) target.
  dataEngineLogFlags: ~
  # -- Настройка замораживания файловой системы на root разделе перед созданием снапшота.
  freezeFilesystemForSnapshot: ~
  # -- Настройка автоматической очистки снапшота при удалении бэкапа.
  autoCleanupSnapshotWhenDeleteBackup: ~
  # -- Настройка автоматической очистки снапшота после завершения on-demand бэкапа.
  autoCleanupSnapshotAfterOnDemandBackupCompleted: ~
  # -- Настройка, позволяющая Longhorn определять падение ноды и немедленно мигрировать затронутые RWX тома.
  rwxVolumeFastFailover: ~
  # -- Включает автоматическую пересборку деградированных реплик пока том отключён. Эта настройка действует только если индивидуальная настройка тома установлена в `ignored` или `enabled`.
  offlineReplicaRebuilding: ~
# -- Настройка, позволяющая обновить backupstore по умолчанию.
defaultBackupStore:
  # -- Endpoint для доступа к backupstore по умолчанию. (Варианты: "NFS", "CIFS", "AWS", "GCP", "AZURE")
  backupTarget: ~
  # -- Имя секрета Kubernetes с credentials для backup target по умолчанию.
  backupTargetCredentialSecret: ~
  # -- Количество секунд ожидания перед проверкой backupstore по умолчанию на наличие новых бэкапов. Значение по умолчанию: "300". При значении "0" polling отключён.
  pollInterval: ~
privateRegistry:
  # -- Установите `true` для автоматического создания секрета приватного реестра.
  createSecret: ~
  # -- URL приватного реестра. Если не указано, Longhorn использует системный реестр по умолчанию.
  registryUrl: ~
  # -- Учётная запись для аутентификации в приватном реестре.
  registryUser: ~
  # -- Пароль для аутентификации в приватном реестре.
  registryPasswd: ~
  # -- Если создание нового секрета true, создать секрет Kubernetes с этим именем; иначе использовать существующий секрет с этим именем. Используется для pull образов из приватного реестра.
  registrySecret: ~
longhornManager:
  log:
    # -- Формат логов Longhorn Manager. (Варианты: "plain", "json")
    format: plain
  # -- PriorityClass для Longhorn Manager.
  priorityClass: *defaultPriorityClassNameRef
  # -- Tolerations для Longhorn Manager на нодах, где разрешены компоненты Longhorn.
  tolerations: []
  ## Если хотите установить tolerations для DaemonSet Longhorn Manager, удалите `[]` в строке выше
  ## и раскомментируйте этот блок примера
  # - key: "key"
  #   operator: "Equal"
  #   value: "value"
  #   effect: "NoSchedule"
  # -- Node selector для Longhorn Manager. Укажите ноды, на которых разрешён Longhorn Manager.
  nodeSelector: {}
  ## Если хотите установить node selector для DaemonSet Longhorn Manager, удалите `{}` в строке выше
  ## и раскомментируйте этот блок примера
  #  label-key1: "label-value1"
  #  label-key2: "label-value2"
  # -- Аннотации для сервиса Longhorn Manager.
  serviceAnnotations: {}
  ## Если хотите установить аннотации для сервиса Longhorn Manager, удалите `{}` в строке выше
  ## и раскомментируйте этот блок примера
  #  annotation-key1: "annotation-value1"
  #  annotation-key2: "annotation-value2"
longhornDriver:
  log:
    # -- Формат логов longhorn-driver. (Варианты: "plain", "json")
    format: plain
  # -- PriorityClass для Longhorn Driver.
  priorityClass: *defaultPriorityClassNameRef
  # -- Tolerations для Longhorn Driver на нодах, где разрешены компоненты Longhorn.
  tolerations: []
  ## Если хотите установить tolerations для Deployment Longhorn Driver Deployer, удалите `[]` в строке выше
  ## и раскомментируйте этот блок примера
  # - key: "key"
  #   operator: "Equal"
  #   value: "value"
  #   effect: "NoSchedule"
  # -- Node selector для Longhorn Driver. Укажите ноды, на которых разрешён Longhorn Driver.
  nodeSelector: {}
  ## Если хотите установить node selector для Deployment Longhorn Driver Deployer, удалите `{}` в строке выше
  ## и раскомментируйте этот блок примера
  #  label-key1: "label-value1"
  #  label-key2: "label-value2"
longhornUI:
  # -- Количество реплик Longhorn UI.
  replicas: 2
  # -- PriorityClass для Longhorn UI.
  priorityClass: *defaultPriorityClassNameRef
  # -- Affinity для подов Longhorn UI. Укажите желаемую affinity для Longhorn UI.
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - longhorn-ui
            topologyKey: kubernetes.io/hostname
  # -- Tolerations для Longhorn UI на нодах, где разрешены компоненты Longhorn.
  tolerations: []
  ## Если хотите установить tolerations для Deployment Longhorn UI, удалите `[]` в строке выше
  ## и раскомментируйте этот блок примера
  # - key: "key"
  #   operator: "Equal"
  #   value: "value"
  #   effect: "NoSchedule"
  # -- Node selector для Longhorn UI. Укажите ноды, на которых разрешён Longhorn UI.
  nodeSelector: {}
  ## Если хотите установить node selector для Deployment Longhorn UI, удалите `{}` в строке выше
  ## и раскомментируйте этот блок примера
  #  label-key1: "label-value1"
  #  label-key2: "label-value2"
ingress:
  # -- Настройка, позволяющая Longhorn создавать записи ingress для сервиса Longhorn UI.
  enabled: false
  # -- Ресурс IngressClass, содержащий конфигурацию ingress, включая имя Ingress контроллера.
  # ingressClassName может заменить аннотацию kubernetes.io/ingress.class из ранних версий Kubernetes.
  ingressClassName: ~
  # -- Hostname балансировщика нагрузки Layer 7.
  host: sslip.io
  # -- Настройка, позволяющая включить TLS на записях ingress.
  tls: false
  # -- Настройка, позволяющая включить безопасные соединения к сервису Longhorn UI через порт 443.
  secureBackends: false
  # -- TLS секрет с приватным ключом и сертификатом для TLS. Применяется только при включённом TLS на ingress.
  tlsSecret: longhorn.local-tls
  # -- Путь ingress по умолчанию. Доступ к Longhorn UI по полному пути {{host}}+{{path}}.
  path: /
  # -- Тип пути ingress. Для обратной совместимости значение по умолчанию: "ImplementationSpecific".
  pathType: ImplementationSpecific
  ## Если используете kube-lego, добавьте:
  ## kubernetes.io/tls-acme: true
  ##
  ## Полный список аннотаций ingress:
  ## ref: https://github.com/kubernetes/ingress-nginx/blob/master/docs/annotations.md
  ##
  ## Если tls установлен в true, аннотация ingress.kubernetes.io/secure-backends: "true" добавляется автоматически
  # -- Аннотации ingress в формате key-value.
  annotations:
  #  kubernetes.io/ingress.class: nginx
  #  kubernetes.io/tls-acme: true

  # -- Секрет с TLS приватным ключом и сертификатом. Используйте secrets если хотите свои сертификаты для защиты ingress.
  secrets:
  ## Если предоставляете свои сертификаты, используйте это для добавления сертификатов как secrets
  ## key и certificate должны начинаться с -----BEGIN CERTIFICATE----- или
  ## -----BEGIN RSA PRIVATE KEY-----
  ##
  ## name должен соответствовать tlsSecret выше
  ## Если используете kube-lego, это не нужно - он создаст секрет автоматически
  ##
  ## Также можно создавать и управлять сертификатами вне этого helm chart
  ## См. README.md для подробностей
  # - name: longhorn.local-tls
  #   key:
  #   certificate:
# -- Настройка, позволяющая включить pod security policies (PSPs) для привилегированных подов Longhorn. Применяется только к кластерам Kubernetes 1.25 и ранее с включённым встроенным Pod Security admission controller.
enablePSP: false
# -- Переопределение namespace, полезно при использовании longhorn как sub-chart когда namespace релиза не `longhorn-system`.
namespaceOverride: ""
# -- Аннотации для подов DaemonSet Longhorn Manager. Опционально.
annotations: {}
serviceAccount:
  # -- Аннотации для service account
  annotations: {}
metrics:
  serviceMonitor:
    # -- Настройка, позволяющая создать ресурс Prometheus ServiceMonitor для компонентов Longhorn Manager.
    enabled: false
    # -- Дополнительные labels для ресурса Prometheus ServiceMonitor.
    additionalLabels: {}
    # -- Аннотации для ресурса Prometheus ServiceMonitor.
    annotations: {}
    # -- Интервал сбора метрик Prometheus с target.
    interval: ""
    # -- Таймаут после которого Prometheus считает сбор неудачным.
    scrapeTimeout: ""
    # -- Правила relabeling для применения labels метаданных target. См. [документацию Prometheus Operator](https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.Endpoint) для форматирования.
    relabelings: []
    # -- Правила relabeling для применения к samples перед сохранением. См. [документацию Prometheus Operator](https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.Endpoint) для форматирования.
    metricRelabelings: []
## Настройки OpenShift
openshift:
  # -- Настройка, позволяющая Longhorn интегрироваться с OpenShift.
  enabled: false
  ui:
    # -- Route для соединений между Longhorn и web console OpenShift.
    route: "longhorn-ui"
    # -- Порт для доступа к web console OpenShift.
    port: 443
    # -- Порт для proxy доступа к web console OpenShift.
    proxy: 8443
# -- Настройка, позволяющая Longhorn генерировать профили code coverage.
enableGoCoverDir: false
# -- Добавить дополнительные манифесты объектов
extraObjects: []
