# Миграция хостов {{ CH }}-кластера в другую зону доступности

Хосты кластера {{ mch-name }} располагаются в [зонах доступности](../../overview/concepts/geo-scope.md) {{ yandex-cloud }}. Хосты можно перенести из одной зоны в другую.

{% note warning %}

{{ mch-name }} поддерживает миграцию только [реплицируемых таблиц](../concepts/replication.md#replicated-tables) на движке семейства `ReplicatedMergeTree`. Если таблицы не реплицируемые, они будут потеряны во время миграции.

{% endnote %}

Чтобы перенести хосты кластера из одной зоны доступности в другую:

1. [Создайте подсеть](../../vpc/operations/subnet-create.md) в зоне доступности, в которую вы переносите хосты.
1. Добавьте хост в кластер:

   {% list tabs group=instructions %}

   - Консоль управления {#console}

      1. Перейдите на [страницу каталога]({{ link-console-main }}) и выберите сервис **{{ ui-key.yacloud.iam.folder.dashboard.label_managed-clickhouse }}**.
      1. Нажмите на имя нужного кластера и перейдите на вкладку **{{ ui-key.yacloud.mdb.cluster.hosts.label_title }}**.
      1. Нажмите кнопку ![image](../../_assets/console-icons/plus.svg) **{{ ui-key.yacloud.mdb.cluster.hosts.button_add-host }}**.
      1. Укажите параметры хоста:

         * Зону доступности, куда переносятся хосты.
         * Новую подсеть.
         * Выберите опцию **{{ ui-key.yacloud.mdb.hosts.dialog.field_public_ip }}**, если хост должен быть доступен извне {{ yandex-cloud }}.

      1. Нажмите **{{ ui-key.yacloud.mdb.hosts.dialog.button_choose }}**.

   - CLI {#cli}

      {% include [cli-install](../../_includes/cli-install.md) %}

      {% include [default-catalogue](../../_includes/default-catalogue.md) %}

      Выполните команду:

      ```bash
      {{ yc-mdb-ch }} host add \
         --cluster-name <имя_кластера> \
         --host type=clickhouse,`
               `zone-id=<зона_доступности>,`
               `subnet-id=<идентификатор_новой_подсети>,`
               `assign-public-ip=<публичный_доступ_к_хосту>
      ```

      Где `assign-public-ip` — публичный доступ к хосту: `true` или `false`.

      Имя кластера можно получить со [списком кластеров в каталоге](cluster-list.md#list-clusters). В параметре `zone-id` укажите зону, куда вы переносите хосты.

   - {{ TF }} {#tf}

      1. В конфигурационный файл {{ TF }} с планом инфраструктуры добавьте манифест хоста:

         ```hcl
         resource "yandex_mdb_clickhouse_cluster" "<имя_кластера>" {
           ...
           host {
             type             = "CLICKHOUSE"
             zone             = "<зона_доступности>"
             subnet_id        = "<идентификатор_новой_подсети>"
             assign_public_ip = <публичный_доступ_к_хосту>
           }
         }
         ```

         Где `assign_public_ip` — публичный доступ к хосту: `true` или `false`.

         В параметре `zone` укажите зону, куда вы переносите хосты.

      1. Проверьте корректность настроек.

         {% include [terraform-validate](../../_includes/mdb/terraform/validate.md) %}

      1. Подтвердите изменение ресурсов.

         {% include [terraform-apply](../../_includes/mdb/terraform/apply.md) %}

   - API {#api}

      Чтобы добавить хост в кластер, воспользуйтесь методом REST API [addHosts](../api-ref/Cluster/addHosts.md) для ресурса [Cluster](../api-ref/Cluster/index.md) или вызовом gRPC API [ClusterService/AddHosts](../api-ref/grpc/cluster_service.md#AddHosts) и передайте в запросе:

      * Идентификатор кластера в параметре `clusterId`. Идентификатор можно получить со [списком кластеров в каталоге](cluster-list.md#list-clusters).
      * Настройки нового хоста в параметрах `hostSpecs`.

   {% endlist %}

1. Чтобы успешно выполнять подключение к базе данных после миграции, укажите FQDN нового хоста в вашем бэкенде или клиенте (например, в коде или графической IDE). Удалите FQDN прежнего хоста в первоначальной зоне.

   Чтобы узнать FQDN, получите список хостов в кластере:

   ```bash
   {{ yc-mdb-ch }} host list --cluster-name <имя_кластера>
   ```

   FQDN указан в выводе команды, в столбце `NAME`. Также вы можете использовать [особый FQDN](connect.md#special-fqdns) для подключения.

1. Удалите хосты в первоначальной зоне доступности:

   {% list tabs group=instructions %}

   - Консоль управления {#console}

      1. Перейдите на [страницу каталога]({{ link-console-main }}) и выберите сервис **{{ ui-key.yacloud.iam.folder.dashboard.label_managed-clickhouse }}**.
      1. Нажмите на имя нужного кластера и выберите вкладку **{{ ui-key.yacloud.mdb.cluster.hosts.label_title }}**.
      1. Нажмите на значок ![image](../../_assets/console-icons/ellipsis.svg) в строке нужного хоста, выберите пункт **{{ ui-key.yacloud.common.delete }}** и подтвердите удаление.

   - CLI {#cli}

      Выполните команду для каждого хоста:

      ```bash
      {{ yc-mdb-ch }} host delete <FQDN_хоста> --cluster-name <имя_кластера>
      ```

   - {{ TF }} {#tf}

      1. В конфигурационном файле {{ TF }} с планом инфраструктуры удалите из описания кластера блоки `host` с первоначальной зоной доступности.
      1. Проверьте корректность настроек.

         {% include [terraform-validate](../../_includes/mdb/terraform/validate.md) %}

      1. Введите слово `yes` и нажмите **Enter**.

         {% include [terraform-apply](../../_includes/mdb/terraform/apply.md) %}

   - API {#api}

      Чтобы удалить хост, воспользуйтесь методом REST API [deleteHosts](../api-ref/Cluster/deleteHosts.md) для ресурса [Cluster](../api-ref/Cluster/index.md) или вызовом gRPC API [ClusterService/DeleteHosts](../api-ref/grpc/cluster_service.md#DeleteHosts) и передайте в запросе:

      * Идентификатор кластера в параметре `clusterId`. Чтобы узнать идентификатор, [получите список кластеров в каталоге](cluster-list.md#list-clusters).
      * FQDN-имя или массив имен удаляемых хостов в параметре `hostNames`.

   {% endlist %}

1. Дождитесь, когда кластер перейдет в состояние **Alive**. В консоли управления перейдите на страницу каталога и выберите сервис **{{ ui-key.yacloud.iam.folder.dashboard.label_managed-clickhouse }}**. Состояние кластера отображается в столбце **{{ ui-key.yacloud.mdb.clusters.column_availability }}**.

{% include [zone-d-restrictions](../../_includes/mdb/ru-central1-d-restrictions.md) %}

{% include [migration-in-data-transfer](../../_includes/data-transfer/migration-in-data-transfer.md) %}

{% include [clickhouse-disclaimer](../../_includes/clickhouse-disclaimer.md) %}
