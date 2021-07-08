Trendyol Case

1.1- Kubernetes Kurulumu
Bu kurulum icin 2 Server kullanılacak; 1 Master,Node rolü olacak. (Server 0) 1 Node (Server 1)
Kurulum yapılırken base-setup olarak aşağıdaki şartlar sağlanmalıdır. 

cluster ismi: case-<ad-soyad>.abc
network plugin: flannel
dns mode: coredns

```ansible-playbook site.yml -i hosts  -l k8s_group  --tags "k8s-install" -vv --flush-cache ```

    Kurulumlar  için ansible kullandım.
    Kubernetes kurulumu için role oluşturdum uzun bir kurulum olacağında içinde bunu tasklere böldüm.
    1. ve 2.  Task: --> bunlar değişkenlerimi almak için kullndım.
    3. Task: --> Hem node hemde masterda kubernetes kurulumu için gereki olan adımları içeriyor.
    4. Task: --> Burada master kurulumunu tamamladım.bu kurulumu kubeadm ile config file ile yaptım bende istene değerleri  burada   belirttim.Core DNS kullanacağımı ve cluster ismimi. aynı zamanda podsubnet ve api nin apisinide burada belirrtim docker    değil containerd kullanacağım için burada onuda belirttim.
    5. Task: --> Burada flannel network kurulumu için gerekli yamlları yükledim.Sunucuyı restart edip hazır hale gelmesi için   beklettim.
    6. Task -->  Burada masterında worker node gibi çalışması için master untaint edildi.(kubectl taint nodes --all node-role.kubernetes.io/master-) - node ekleebilmesi içn gerekn tokenları variable olarak ekledim.
    7. Task -->  Burada node rolindeki server clustera eklendi.

````
    NAME                  STATUS   ROLES                         AGE   VERSION
    server-tugce-esen-0   Ready    control-plane,master,worker   9h    v1.21.2
    server-tugce-esen-1   Ready    worker                        8h    v1.21.2
````

````
    Kubernetes control plane is running at https://192.168.6.97:6443
    CoreDNS is running at https://192.168.6.97:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
````

    ---> Kontrol konusunda eksik bir yapı örneğin aslında kurulumlardan önce birde kontrol mekanızması koymam lazım bu aslında  var mı gibi .Zaman nedeni ile  şuan kotrolleri yok.      



1.2- Kurulum tamamlandıktan sonra cluster üzerinde metric toplamak için internal prometheus-server kurulması ve sadece belirlenen tek bir node üzerinde çalışması sağlanması.

````
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "kube-state-metrics-install" -vv --flush-cache
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "prometheus-install" -vv --flush-cache

````


    Bu iş için iki farklı role ile çalışıyorum.Önce prometheus  metricleri alabilmesi için kubernetes-state-metrics kurulumu yapacağım(kolaya kaçtım zaman nedeni ile apiden de kendim çekebileceğim şeyler vardı.)
    Kurulumları basic yaml'lar ile yaptım.
    Prometheus içinse  sonradan ekleme yapacağımdan birde tags'ler ekeledim prometheus config ayarlarını  değiştirmem gerektiğinde. tags belirterek install için kullandığım diğer kontrollerden kurtulmuş olacağım. 
    Prometheus config dosyası config mapten çekiyor.Değişimleri değişkenleri oarada kullandım.
    Prometheus'a öncelikle nodeselector koyarak belli bir nodeda çalışmasını sağladım.

````  nodeSelector:
        kubernetes.io/hostname: {{ permission_prometheus }}
````
    Prometheus için kurulum yaplırken hangi node'da çalışacağını değişken olarak verdim.(role dizinindeki vars altından güncelliyorum)


1.3- Sadece Prometheus'un çalışması sağlanan o node üzerinde başka bir deployment'ın çalışmaması sağlanması.

````
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "restrict_deployment" -vv --flush-cache
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "prometheus-edit" -vv --flush-cache
````
    Burada da prometheusun çalıştığı node da başaka bir deployment olmasını engellemek için " kullandım.
    Gene aynı şekilde {{ permission_prometheus }} değişkeni ile belirledim.
    Bunu yaptıktan sonra prometheus depolynmetına'da "tolerations girildi deployment editlendi.(restart olursa o padda ayağa kalkabilmesi için.)


1.4- Ingress Controller DaemonSet kurulumu yapılarak; http 80 portu üzerinden deploy edilen prometheus servis erişiminin sağlanması.

``` 
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "ingress-install" -vv --flush-cache
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "prometheus-ingress" -vv --flush-cache
```

    Gene yaml'lar ile ingress kurulumu yapıldı.Burada externalIP kullandım ingress service için. 
    Herhangi bir variable girmemize gerek yok ipleri fact ile alıyorum burada.
    Prometheus için gene prometheus rolünde prometheus-ingress olarak taglediğim task'ler ile kurulumları bitiriyorum.
    Domain name olarak "prometheus.esentu.com" ile hizmet vermesini istiyorum. 
    Burada ilk başta dıaşrdan da erişimi olsun diye anladığım için bu şekilde yaptım.

2. - Verilen sunuculardan Server 2 üzerine Server mode bir Consul kurulumu. Consul cluster tek node olacak şekilde kurulmalı.
    
```` 
    ansible-playbook site.yml -i hosts  -l consul_server  --tags "consul-server-install" -vv --flush-cache

````
    Kurulum ansible ila yapıldı.
    Centos için paketi vardı paket ile kurulum yapıldı.
    Tek node'lu cluster olarak çalışabilmesi için "server = true" olmalı yoksa direk agent olarak başaltıyor servis ve  "bootstrap_expect = 1" parametresinin 1 ayarlanması lazım yoksa lidersiz kalıyor. UI  congfigini açtım.

3. - Server 2 üzerine Federation Prometheus kurulması.
    
```
ansible-playbook site.yml -i hosts  -l prometheus_federation_server  --tags "prometheus-federations-install" -vv --flush-cache

```
3.1 Federation işlemi yapılırken prometheus.yaml’da consul-prometheus discovery kullanılması.
Bu işlemin yapılabilmesi için internal prometheus’un service bilgilerinin Consul’e service registry yapılması gerekmektedir.

````
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "k8s-install-consul-agent" -vv --flush-cache
    ansible-playbook site.yml -i hosts  -l prometheus_federation_server  --tags "prometheus-federations-edit" -vv --flush-cache
````

    Öncelikle servislerin discovery olabilmesi için k8s' helm chart kullanarak consul agent kurdum.
    Config ayarlarınıda config map ile verdim.
    Serviceler register olduktan sonra base prometheus kurulumu yaparak.Config ayarından gitmesi gereken child prometheusu      dinamik olarak almasını sağladım.

```
    consul_sd_configs:
    - server: '{{ ansible_default_ipv4.address }}:8500' --> consul sunucusu
      services: [ 'prometheus-service-monitoring' ]     --> consula register olan izleyeceğimiz servisin adı

    relabel_configs:
    - source_labels: ['__meta_consul_address'].  --> bu şekilde register olan child prometheusun adresi alınır. 
      separator: ':'
      target_label:  'instance'
      replacement: '${1}:30100'
      action: 'replace'

```

4. Server 2 üzerine veya kubernetes içine Grafana kurulması ve external Prometheus datasource eklenmesi.

4.1 Kurulan kubernetes cluster’ı için aşağıdaki memory metric’lerini içeren bir dashboard hazırlanması.

```
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "k8s-grafana-install" -vv --flush-cache
    ansible-playbook site.yml -i hosts  -l k8s_group  --tags "grafana-add" -vv --flush-cache
```

    Kubernets içine grafana kurulumu yaptım. Ingress açarak "grafana.esentu.com" ile de girişi sağladım.
    Öncelikle başka bir grafanada kurulumları arayüzden yaptım daha sonra api ile json formataında templatelerini aldım.
    Aldığım templateleri grafana apisi ile json formatındaki sırası ile data source ve dachboard olarak ekledim.
 
    datasoruce api:/api/datasources
    dashboard api:/api/dashboards/db


5. Server 2 üzerine Alertmanager kurulması ve 1.2’de deploy edilen uygulama için pod_restart alarm’i olusturulması.

``` 
    ansible-playbook site.yml -i hosts  -l alertmanager_servers   --tags "alertmanager-install" -vv --flush-cache
    ansible-playbook site.yml -i hosts  -l alertmanager_servers   --tags "alert-add" -vv --flush-cache
```

    Alertmanager için kullanıcı, dizin ve servis yaratıldı.Kurulum yapıldı.
    Alertmanager kurulduktan sonra prometheus confuna eklendi ve prometheus-edit ile bu işte tamamlandı.
    pod_restart için ise rule olusturdu rule: son 90sn deki prometheus'un  restart sayısın rateine bakıyor.

``` 
    rate(kube_pod_container_status_restarts_total{container="prometheus"}[90s]) > 0  
```

6. Server 3 üzerine Elasticsearch ve Kibana kurulmasi ve kubernetes podlarının stdout loglarının buraya yönlendirilmesi.

``` 
    ansible-playbook site.yml -i hosts  -l elasticsearch_servers   --tags "elasticsearch-install" -vv --flush-cache 
    ansible-playbook site.yml -i hosts  -l elasticsearch_servers   --tags "kibana-install" --flush-cache -vv
    ansible-playbook site.yml -i hosts  -l k8s_group   --tags "fluentbit-install" -vv --flush-cache

```

    Elasticsearch ve kibana kurulumu içn kendim daha önceden (6.x) kullandığım bir ansible vardı onu modifiye ettim yeni    versiyonu ile kurulumlarını yaptım.
    Kubernets podlarının datasını çekmek içinse filebit kullandım.
    Burada gene containerd kullandığım için bir kaç değişiklik yapmam grekiyordu logları farklı yerlere yazıyor ve log formatları farklı. Parse içinde docker için yazılan kullanılamıyor.

    docker için /var/lib/docker/container altını kontrol ederken burası containerd için yok.
    /var/log/pod ve /var/log/containers altına bakması lazım.

    filebit ayarlarında ise (config mapten alıyor)
 

```
     [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri                     --> parserın docker değil cri seçilmesi lzım yada siz ne ismlendirme aparsanız.
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [PARSER]
        # http://rubular.com/r/tjUt3Awgg4
        Name cri
        Format regex
        Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

7. Server3 üzerine bir Gitlab kurularak 3. ve 5. maddelerin pipeline üzerinden tetiklenebiliyor olması.

```
    ansible-playbook site.yml -i hosts  -l gitlab_servers   --tags "gitlab" --flush-cache -vv

```

    Ansible üzrinde gitlab kurulumu gerçekleştirildi.
    3- madde için  için cı-cd ile tetiklenebilir job oluşturuldu.Gitlab runner da docker olarak kullnadık.onun içinde ansible     yamllarımı çalıştırdım.Bunlar zaten birbirine bağlı maddelerdi sırası ile aynı jobda tekileniyorlar.
    5- maddesi içinde 2 stage ile işlem bitiyor.


    
