
VCF Import Tool 工具使用两种方式来帮助客户将现有的 vSphere 或 vSphere \+ vSAN 环境转变为 VMware Cloud Foundation 环境，分别是**转换（Convert）**和**导入（Import）**。之前在这篇（[使用 VCF Import Tool 将现有 vSphere 环境转换为管理域。](https://github.com)）文章中演示了将现有 vSphere 环境转换为 VCF 解决方案中管理工作负载域的过程，因此这篇文章接着这个主题，看看如何使用这个工具将现有 vSphere 环境导入为 VCF 解决方案中的 VI 工作负载域。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240928122908110-980062923.png)](https://github.com)


 



## 一、使用要求和限制



请注意，使用 VCF Import Tool 工具执行导入（Import）过程与之前的转换（Convert）过程具有相同的[使用 VCF Import Tool 将现有 vSphere 环境转换为管理域。](https://github.com)）中的说明，这里不再赘述。


使用要求和限制中有一条是，现有 vSphere 环境的 vCenter Server 虚拟机要么位于自身的集群内要么运行在 VCF 管理域集群上，当在执行导入（Import）任务的同时进行 NSX 方案的部署，则会出现两种不同的情况：


* 如果 vCenter Server 虚拟机运行在自身 vSphere 集群内，当执行导入任务并同时部署 NSX 方案，则 NSX Manager 虚拟机将部署到 vSphere 集群内。
* 如果 vCenter Server 虚拟机运行在 VCF 管理域集群上，当执行导入任务并同时部署 NSX 方案，则 NSX Manager 虚拟机将部署到 VCF 管理域集群上。


 



## 二、现有 vSphere 环境



针对 VCF Import Tool 工具使用的各种要求和限制，同样需要对现有 vSphere 环境的信息进行检查确认。本次用于导入（Import）的现有 vSphere 环境如下图所示，一个由 3 主机所组成的 vSAN ESA 标准集群，关于具体检查过程这里就略过了，详细请参看之前文章。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240925163330604-1135753176.png)](https://github.com)


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240925163400538-1037152474.png)](https://github.com)


当前 vSphere 环境的 vCenter Server 虚拟机运行在 VCF 管理域集群上（注意，后面有调整）。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240925163722548-1700141461.png)](https://github.com)


 



## 三、执行导入前预检查



在对现有 vSphere 环境执行导入（Import）操作之前，需要在 SDDC Manager 上执行一次预检查，确定当前 vSphere 环境是否满足导入为 VI 域的要求。由于之前在进行转换（Convert）时，已经将 VCF Import Tool 工具包上传至 SDDC Manager 了，所以这里无需进行上传。通过 SSH 以 vcf 用户连接到 SDDC Manager 的命令行，使用以下命令进入到 vcf\-brownfield\-toolset 目录。



```
cd /home/vcf/vcfimport/vcf-brownfield-import-5.2.0.0-24108578/vcf-brownfield-toolset/
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240925164248933-1955789235.png)](https://github.com)


进入目录后，使用以下命令在 SDDC Manager 上执行环境检查。总共有 83 个内容，成功检查 82 个，失败 0 个，内部错误 1 个。



```
python3 vcf_brownfield.py check --vcenter vcf-vi01-vcsa01.mulab.local --sso-user administrator@vsphere.local --sso-password Vcf520@password
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240925164949944-992589903.png)](https://github.com)


根据检查输出的结果，查看具体错误的内容，Compatibility validation 项检查错误，如下所示。



```
An error occurred when validating VMware Cloud Foundation compatibility: File with Compatibility Matrix Content for Compatibility controller VMWARE_COMPAT is not found.
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240925165048400-952241306.png)](https://github.com)


这个错误必须要解决，否则后面无法继续。一开始的时候，我一直以为是 vSAN HCL 文件有问题导致的，经过各种尝试和验证，还把之前转换的管理域给清除了，重新部署了 VCF 管理域环境，结果这个错误始终无法得到修复。然后后面才反应过来，它这个错误指的是“Compatibility Matrix”兼容性数据文件无法验证，这是一个专门针对的 VCF 的兼容性矩阵数据，跟 vSAN 那个 HCL 数据库文件有点类似但不一样，也就是这个数据现在缺失了无法进行验证，所以有这个内部错误。其实，你也可以在 SDDC Manager UI 下图中的地方看到这个错误提示，不管是通过转换（Convert）过来的管理域还是使用 Cloud Builder 部署的管理域，这个 VCF 兼容性数据默认都没有，我们需要手动下载并进行更新。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926200345155-2022136300.png)](https://github.com)


如何手动下载并进行更新 VCF 兼容性数据文件，也就是这个“Compatibility Matrix”文件，可以参考这篇[《Offline Download of VMware Cloud Foundation 5\.2 Upgrade Bundles》](https://github.com)产品文档。整个流程大概是，你需要使用 Bundle Transfer Utility 工具下载这个“Compatibility Matrix”兼容性数据文件，然后再使用这个工具将兼容性数据文件更新到 VCF 中。


对于这个工具，可以到 Broadcom 支持门户（BSP）去下载，我已经将这个工具和“Compatibility Matrix”兼容性数据文件上传到了这个百度网盘链接（[https://pan.baidu.com/s/1lUbrN0zjLUUC1oB8L7ZRAg?pwd\=lvx9](https://github.com)）中，有需要可以去下载。下面我将演示如何去下载这个兼容性数据文件并将其更新到 SDDC Manager 中。


如果 VCF 环境可以连接互联网，那可以直接使用这个工具在 SDDC Manager 上执行下载和更新过程，但要是不能联网的话，你可以找一台能够连接互联网的 Linux 主机，然后在这个主机上再使用工具去下载兼容性数据文件（VmwareCompatibilityData.json），下载之后保存到本地，将文件上传到内网环境的 SDDC Manager 上，再使用这个工具完成更新。注意，下载这个兼容性数据文件需要具有 [Broadcom 支持门户](https://github.com)（BSP）的账号，没有 VCF 权限仅普通用户权限也可以进行下载。


将 Bundle Transfer Utility 工具上传到具有互联网连接的 Linux 主机上，解压后并赋予 lcm\-bundle\-transfer\-util 工具执行权限，然后使用以下命令下载“Compatibility Matrix”兼容性数据文件。



```
./lcm-bundle-transfer-util --download --compatibilityMatrix --depotUser xxxxxxxx@163.com
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926200412328-905194873.png)](https://github.com)


获取到 VCF 兼容性数据文件后，将该文件以及 Bundle Transfer Utility 工具一并上传到 VCF 管理域的 SDDC Manager 上，使用以下命令解压该工具并调整权限。



```
mkdir /opt/vmware/vcf/lcm/lcm-tools
cd /opt/vmware/vcf/lcm/
tar -xvf lcm-tools-prod.tar.gz
chown vcf_lcm:vcf -R lcm-tools
chmod 750 -R lcm-tools
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926200453513-579687402.png)](https://github.com)


进入到工具所在的目录，使用以下命令完成 VCF 兼容性数据的更新，请注意命令中“inputDirectory”选项所使用的目录位置。



```
./lcm-bundle-transfer-util --update --compatibilityMatrix --inputDirectory /home/vcf/ --sddcMgrFqdn vcf-mgmt01-sddc01.mulab.local --sddcMgrUser administrator@vsphere.local
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926200506537-1170741640.png)](https://github.com)


完成对“Compatibility Matrix”兼容性数据的更新之后，再次使用以下命令执行环境检查，现在所有检查都已成功。



```
python3 vcf_brownfield.py check --vcenter vcf-vi01-vcsa01.mulab.local --sso-user administrator@vsphere.local --sso-password Vcf520@password
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926202318586-1922787006.png)](https://github.com)


 



## 四、准备 NSX Manager



在进行转换（Convert）时，我们同时执行了 NSX 的部署，在进行导入（Import）时，同样可以一起执行 NSX 的部署。这一步骤是可选操作，你可以在执行导入过程的同时执行 NSX 的部署，也可以在执行导入结束之后，在其他时间再执行 NSX 的部署，只需要使用 \-\-skip\-nsx\-deployment 选项跳过就行。以下是执行 NSX 部署所需的 [JSON 配置文件](https://github.com)。



```
{
  "license_key": "AAAAA-BBBBB-CCCCC-DDDDD-EEEEE",
  "form_factor": "medium",
  "admin_password": "Vcf520@password",
  "install_bundle_path": "/nfs/vmware/vcf/nfs-mount/bundle/bundle-124941.zip",
  "cluster_ip": "192.168.32.76",
  "cluster_fqdn": "vcf-vi01-nsx01.mulab.local",
  "manager_specs": [{
    "fqdn": "vcf-vi01-nsx01a.mulab.local",
    "name": "vcf-vi01-nsx01a",
    "ip_address": "192.168.32.77",
    "gateway": "192.168.32.254",
    "subnet_mask": "255.255.255.0"
  },
  {
    "fqdn": "vcf-vi01-nsx01b.mulab.local",
    "name": "vcf-vi01-nsx01b",
    "ip_address": "192.168.32.78",
    "gateway": "192.168.32.254",
    "subnet_mask": "255.255.255.0"
  },
  {
    "fqdn": "vcf-vi01-nsx01c.mulab.local",
    "name": "vcf-vi01-nsx01c",
    "ip_address": "192.168.32.79",
    "gateway": "192.168.32.254",
    "subnet_mask": "255.255.255.0"
  }]
}
```

将 NSX 部署的 JSON 配置文件以及 NSX 的安装包上传到 SDDC Manager 中，需要记住这个配置文件上传的路径，后面需要用到。



```
ls /home/vcf/vcfimport/
ls /nfs/vmware/vcf/nfs-mount/bundle/
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240918210328590-283524163.png)](https://github.com)


其实，在执行 NSX 部署时，如果物理资源不是很充足的话，我们也可以只部署 1 个 NSX Manager 设备，之前在执行转换（Convert）的时候没有说，以下方法应该也同样适用，有需要的可以参考以下步骤。当然，上面的 JSON 配置文件还是需要配置完整的 3 个 NSX Manager 设备，如果不完整，那后面的检查都过不了。


通过 SSH 连接到 SDDC Manager 并切换到 root 用户，使用以下命令编辑系统配置文件：



```
vim /etc/vmware/vcf/domainmanager/application-prod.properties
```

添加以下内容并保存：



```
nsxt.manager.cluster.size=1
nsxt.manager.formfactor=medium
```

重新启动服务。



```
systemctl restart domainmanager
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926202747158-682989083.png)](https://github.com)


 



## 五、正式执行导入过程



通过 vcf 用户 SSH 登录到 SDDC Manager，进入到 vcf\-brownfield\-toolset 目录，使用以下命令执行 vSphere 环境导入过程。运行命令后，输入 SDDC Manager 的 admin 用户密码以及 vCenter Server 的 root 和 SSO 用户密码进行验证。



```
python3 vcf_brownfield.py import --vcenter vcf-vi01-vcsa01.mulab.local --sso-user administrator@vsphere.local --domain-name vcf-vi01 --nsx-deployment-spec-path /home/vcf/vcfimport/vcf520-import-nsx.json
```

[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926204420957-877860532.png)](https://github.com)


输入 YES 开始导入（Import）过程。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926204554762-873152921.png)](https://github.com)


登录 SDDC Manager UI 跟踪任务状态。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926204634167-541716372.png)](https://github.com)


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926204705626-123132136.png)](https://github.com):[蓝猫机场](https://fenfang.org)


任务执行一段时间后，在部署 NSX 的时候又失败了......。原因是物理主机的 CPU 负载过高导致管理域的一台虚拟 ESXi 主机卡死了，刚好 SDDC Manager 虚拟机也在这上面，看这篇（[使用 esxtop 杀死 ESXi 主机中卡死和不响应的虚拟机。](https://github.com)）文章！


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926220555441-584051554.png)](https://github.com)


管理域 ESXi 主机卡死后，SDDC Manager 虚拟机进行了 HA 切换，等所有服务都运行正常后，通过 SDDC Manager UI 重新启动失败的任务就可以了，因为现有 vSphere 环境的导入任务已经完成了，如下图所示。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926222325826-1456075127.png)](https://github.com)


最终，任务全部执行成功。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926230255700-2055496488.png)](https://github.com)


任务完成后，切换到 root 用户，重新启动一下 SDDC Manager 的所有服务，并使 UI 重新进行初始化。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926230822299-362589480.png)](https://github.com)


 



## 六、验证已导入的 VI 域



导航到 SDDC Manager UI\-\>清单\-\>工作负载域，可以看到现有 vSphere 环境已经被导入为 VI 域，VI 域的名称为我们导入时设置的名称 vcf\-vi01。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926231241118-776735602.png)](https://github.com)


点击进入 VI 域，查看该工作负载域的摘要信息。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926231325527-976216516.png)](https://github.com)


在主机和集群选项卡中，可以看到属于 vSphere 环境中的 ESXi 主机和集群配置信息。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926231348540-1880293412.png)](https://github.com)


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926231404703-1652177363.png)](https://github.com)


执行一下环境预检查，查看各个组件和配置是否都正常。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926231452688-1039470330.png)](https://github.com)


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232108172-1396899660.png)](https://github.com)


在发行版本视图下，VCF 5\.2 版本中现在具有两个工作负载域。 


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232309016-1667470531.png)](https://github.com)


在映像管理视图下，现在具有两个可用的映像。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232335322-508509047.png)](https://github.com)


在网络设置视图下，查看 VI 域的网络池配置信息。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232358415-1436489949.png)](https://github.com)


在对现有 vSphere 环境进行导入检查时，由于不知道错误的原因，VCF 管理域被重新部署了，所以当时将现有 vSphere 环境的 vCenter Server 虚拟机通过跨 vCenter vMotion 迁移到了自身的集群上。由于物理资源负载过高导致 ESXi 4 主机卡死，所以现在是断开状态。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232454418-447687491.png)](https://github.com)


现有 vSphere 环境信息，现在是 VI 域 vCenter Server，虚拟机清单包含 vCenter Server 虚拟机和 1 个 NSX Manager 虚拟机。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232544267-106727.png)](https://github.com)


vCenter Server 虚拟机在现有 vSphere 集群中，所以 NSX Manager 虚拟机被部署到了相同的位置。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232647644-1952742378.png)](https://github.com)


登录 VI 域的 NSX Manager UI（VIP），查看 NSX 系统配置概览。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232726351-921684094.png)](https://github.com)


NSX 集群配置，只有一个 NSX Manager 节点的 NSX 管理集群。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232750934-2014619318.png)](https://github.com)


VI 域 vCenter Server 已作为计算管理器被添加到 NSX 当中。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232906537-1358429897.png)](https://github.com)


VI 域集群中的主机已配置了分布式虚拟端口组（DVPG）的 NSX。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232823380-1600640971.png)](https://github.com)


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926232951427-192624395.png)](https://github.com)


VI 域管理组件在 DFW 排除列表中。


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926233032695-1768644375.png)](https://github.com)


[![](https://img2024.cnblogs.com/blog/2313726/202409/2313726-20240926233052253-1846517958.png)](https://github.com)


