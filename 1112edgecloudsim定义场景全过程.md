使用EdgeCloudSim构建一个场景，主要目的是为了熟悉EdgeCloudSim，所以场景定义的较简单。在目录src/edu/boun/edgecloudsim/applications下创建一个包test_app1来自定义场景，参考了样例来完成。

## 1.定义main方法以运行场景

创建文件MainApp.java，定义main方法，主要通过运行main方法来启动模拟。

main方法中，主要做了这几件事：

1.根据配置文件路径，使用SimSettings加载相应配置

2.根据模拟场景和编排策略（策略的类型及其所代表的含义都是自定义的），运行具体场景，主要过程：

- 初始化CloudSim包
- 创建工厂类对象
- 常见模拟管理器对象
- 开启模拟

3.将日志打印到指定文件夹中

```java
package edu.boun.edgecloudsim.applications.test_app1;

import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Calendar;
import java.util.Date;

import org.cloudbus.cloudsim.Log;
import org.cloudbus.cloudsim.core.CloudSim;

import edu.boun.edgecloudsim.applications.sample_app1.SampleScenarioFactory;
import edu.boun.edgecloudsim.core.ScenarioFactory;
import edu.boun.edgecloudsim.core.SimManager;
import edu.boun.edgecloudsim.core.SimSettings;
import edu.boun.edgecloudsim.utils.SimLogger;
import edu.boun.edgecloudsim.utils.SimUtils;

public class MainApp {

	/**
	 * Creates main() to run this example
	 */
	public static void main(String[] args) {
		//disable console output of cloudsim library
		Log.disable();
		
		//enable console output and file output of this application
		SimLogger.enablePrintLog();
		
		int iterationNumber = 1;
		String configFile = ""; //配置文件config.properties路径
		String outputFolder = ""; //输出文件路径
		String edgeDevicesFile = ""; //边缘设备edge_devices.xml文件
		String applicationsFile = ""; //应用配置applications.xml文件
		if (args.length == 5){
			configFile = args[0];
			edgeDevicesFile = args[1];
			applicationsFile = args[2];
			outputFolder = args[3];
			iterationNumber = Integer.parseInt(args[4]);
		}
		else{
			SimLogger.printLine("Simulation setting file, output folder and iteration number are not provided! Using default ones...");
			configFile = "scripts/test_app1/config/default_config.properties";
			applicationsFile = "scripts/test_app1/config/applications.xml";
			edgeDevicesFile = "scripts/test_app1/config/edge_devices.xml";
			outputFolder = "sim_results/test_app1/ite" + iterationNumber;
		}

		//根据配置文件加载配置
		SimSettings SS = SimSettings.getInstance();
		if(SS.initialize(configFile, edgeDevicesFile, applicationsFile) == false){
			SimLogger.printLine("cannot initialize simulation settings!");
			System.exit(0);
		}
		
		if(SS.getFileLoggingEnabled()){
			SimLogger.enableFileLog();
			SimUtils.cleanOutputFolder(outputFolder);
		}
		
		DateFormat df = new SimpleDateFormat("dd/MM/yyyy HH:mm:ss");
		Date SimulationStartDate = Calendar.getInstance().getTime();
		String now = df.format(SimulationStartDate);
		SimLogger.printLine("Simulation started at " + now);
		SimLogger.printLine("----------------------------------------------------------------------");

		for(int j=SS.getMinNumOfMobileDev(); j<=SS.getMaxNumOfMobileDev(); j+=SS.getMobileDevCounterSize()) //遍历所有移动设备
		{
			for(int k=0; k<SS.getSimulationScenarios().length; k++) //遍历所有模拟场景
			{
				for(int i=0; i<SS.getOrchestratorPolicies().length; i++) //遍历所有编排策略
				{
					String simScenario = SS.getSimulationScenarios()[k];
					String orchestratorPolicy = SS.getOrchestratorPolicies()[i];
					Date ScenarioStartDate = Calendar.getInstance().getTime();
					now = df.format(ScenarioStartDate);

					SimLogger.printLine("Scenario started at " + now);
					SimLogger.printLine("Scenario: " + simScenario + " - Policy: " + orchestratorPolicy + " - #iteration: " + iterationNumber);
					SimLogger.printLine("Duration: " + SS.getSimulationTime()/60 + " min (warm up period: "+ SS.getWarmUpPeriod()/60 +" min) - #devices: " + j);
					SimLogger.getInstance().simStarted(outputFolder,"SIMRESULT_" + simScenario + "_"  + orchestratorPolicy + "_" + j + "DEVICES");
					
					try
					{
						// 第一步：初始化CloudSim包，应该在创建任何实体之前调用
						int num_user = 2;   // number of grid users
						Calendar calendar = Calendar.getInstance();
						boolean trace_flag = false;  // mean trace events
				
						// Initialize the CloudSim library
						CloudSim.init(num_user, calendar, trace_flag, 0.01);
						
						// 创建场景工厂类，根据需要定义工厂类
						ScenarioFactory sampleFactory = new TestScenarioFactory(j,SS.getSimulationTime(), orchestratorPolicy, simScenario);
						
						// 创建模拟管理器
						SimManager manager = new SimManager(sampleFactory, j, simScenario, orchestratorPolicy);
						
						// 开始模拟
						manager.startSimulation();
					}
					catch (Exception e)
					{
						SimLogger.printLine("The simulation has been terminated due to an unexpected error");
						e.printStackTrace();
						System.exit(0);
					}
					
					Date ScenarioEndDate = Calendar.getInstance().getTime();
					now = df.format(ScenarioEndDate);
					SimLogger.printLine("Scenario finished at " + now +  ". It took " + SimUtils.getTimeDifference(ScenarioStartDate,ScenarioEndDate));
					SimLogger.printLine("----------------------------------------------------------------------");
				}//End of orchestrators loop
			}//End of scenarios loop
		}//End of mobile devices loop

		Date SimulationEndDate = Calendar.getInstance().getTime();
		now = df.format(SimulationEndDate);
		SimLogger.printLine("Simulation finished at " + now +  ". It took " + SimUtils.getTimeDifference(SimulationStartDate,SimulationEndDate));
	}
}

```

## 2.定义工厂类

工厂类主要用于创建所需类的实例，需要重写相应的获取实例方法，包括负载生成模型、边缘编排器、移动模型、网络模型、边缘服务器管理类、云服务器管理、移动设备管理类、移动服务器管理类，这些类都有个默认的实现。

```java
package edu.boun.edgecloudsim.applications.test_app1;

import edu.boun.edgecloudsim.cloud_server.CloudServerManager;
import edu.boun.edgecloudsim.cloud_server.DefaultCloudServerManager;
import edu.boun.edgecloudsim.core.ScenarioFactory;
import edu.boun.edgecloudsim.edge_client.DefaultMobileDeviceManager;
import edu.boun.edgecloudsim.edge_client.MobileDeviceManager;
import edu.boun.edgecloudsim.edge_client.mobile_processing_unit.DefaultMobileServerManager;
import edu.boun.edgecloudsim.edge_client.mobile_processing_unit.MobileServerManager;
import edu.boun.edgecloudsim.edge_orchestrator.BasicEdgeOrchestrator;
import edu.boun.edgecloudsim.edge_orchestrator.EdgeOrchestrator;
import edu.boun.edgecloudsim.edge_server.DefaultEdgeServerManager;
import edu.boun.edgecloudsim.edge_server.EdgeServerManager;
import edu.boun.edgecloudsim.mobility.MobilityModel;
import edu.boun.edgecloudsim.mobility.NomadicMobility;
import edu.boun.edgecloudsim.network.MM1Queue;
import edu.boun.edgecloudsim.network.NetworkModel;
import edu.boun.edgecloudsim.task_generator.IdleActiveLoadGenerator;
import edu.boun.edgecloudsim.task_generator.LoadGeneratorModel;

public class TestScenarioFactory implements ScenarioFactory{
	private int numOfMobileDevice;
	private double simulationTime;
	private String orchestratorPolicy;
	private String simScenario;
	
	TestScenarioFactory(int _numOfMobileDevice,
			double _simulationTime,
			String _orchestratorPolicy,
			String _simScenario){
		orchestratorPolicy = _orchestratorPolicy;
		numOfMobileDevice = _numOfMobileDevice;
		simulationTime = _simulationTime;
		simScenario = _simScenario;
	}
	
	@Override
	public LoadGeneratorModel getLoadGeneratorModel() {
		return new IdleActiveLoadGenerator(numOfMobileDevice, simulationTime, simScenario);
	}

	@Override
	public EdgeOrchestrator getEdgeOrchestrator() {
		return new TestEdgeOrchestrator(orchestratorPolicy, simScenario);
	}

	@Override
	public MobilityModel getMobilityModel() {
		return new NomadicMobility(numOfMobileDevice,simulationTime);
	}

	@Override
	public NetworkModel getNetworkModel() {
		return new MM1Queue(numOfMobileDevice, simScenario);
	}

	@Override
	public EdgeServerManager getEdgeServerManager() {
		return new DefaultEdgeServerManager();
	}

	@Override
	public CloudServerManager getCloudServerManager() {
		return new DefaultCloudServerManager();
	}
	
	@Override
	public MobileDeviceManager getMobileDeviceManager() throws Exception {
		return new DefaultMobileDeviceManager();
	}

	@Override
	public MobileServerManager getMobileServerManager() {
		return new DefaultMobileServerManager();
	}

}

```

## 3.定义移动模块

通过继承MobilityMobility类就可以自定义移动类。

构造函数传入两个参数，移动设备的数目和模拟时间。

重写initalize方法，根据需要初始化代表设备位置的map，key是时间，value是位置，所有的map在一个列表中，列表下标代表了设备号。这是一种方法，并不需要在初始化时就定义好设备在所有时间所在的位置，根据需要来定义即可。

重写getLocation方法，根据设备id和时间，来获取设备当前处于的位置。

#### NomadicMobility

NomadicMobility是默认的移动类。NomadicMobility定义了一种移动设备位置随时间变化的场景。每个移动设备，初始时在任意数据中心的位置。每隔一段时间，会移动到任意一个数据中心的位置。对于每个数据中心，这个间隔的时间，是一个均值为t的指数分布。t是通过配置文件定义，edge_devices.xml文件定义吸引力等级（attractiveness），config.properties文件定义吸引力等级对应的等待时间，也就是这里说的t。

移动模块就使用默认的实现。

## 4.定义边缘编排器

边缘编排器决定了一个任务在哪个设备的哪台虚拟机上运行，继承EdgeOrchestrator抽象类可以定义自己的边缘编排器。构造函数的参数包括编排策略和场景。需要重写的方法包括：initialize初始化，getDeviceToOffload根据任务决定哪台设备来执行卸载，getVmToOffload根据任务和设备来决定哪台虚拟机来执行卸载。这里的卸载是对offload的翻译，我的理解是offload就是处理的意思。

```java
package edu.boun.edgecloudsim.applications.test_app1;

import java.util.List;

import org.cloudbus.cloudsim.Vm;
import org.cloudbus.cloudsim.core.CloudSim;
import org.cloudbus.cloudsim.core.SimEvent;

import edu.boun.edgecloudsim.core.SimManager;
import edu.boun.edgecloudsim.core.SimSettings;
import edu.boun.edgecloudsim.edge_client.CpuUtilizationModel_Custom;
import edu.boun.edgecloudsim.edge_client.Task;
import edu.boun.edgecloudsim.edge_client.mobile_processing_unit.MobileVM;
import edu.boun.edgecloudsim.edge_orchestrator.EdgeOrchestrator;
import edu.boun.edgecloudsim.edge_server.EdgeVM;
import edu.boun.edgecloudsim.utils.SimLogger;

public class TestEdgeOrchestrator extends EdgeOrchestrator{
	private int numberOfHost; //used by load balancer

	public TestEdgeOrchestrator(String _policy, String _simScenario) {
		super(_policy, _simScenario);
	}

	@Override
	public void initialize() {
		numberOfHost=SimSettings.getInstance().getNumOfEdgeHosts();
	}

	/*
	 * (non-Javadoc)
	 * @see edu.boun.edgecloudsim.edge_orchestrator.EdgeOrchestrator#getDeviceToOffload(edu.boun.edgecloudsim.edge_client.Task)
	 * 
	 */
	@Override
	public int getDeviceToOffload(Task task) {
		int result = 0;

		if(policy.equals("TEST_POLICIES")){
			result = SimSettings.GENERIC_EDGE_DEVICE_ID;
		}
		else {
			SimLogger.printLine("Unknow edge orchestrator policy! Terminating simulation...");
			System.exit(0);
		}

		return result;
	}

	@Override
	public Vm getVmToOffload(Task task, int deviceId) {
		Vm selectedVM = null;
		
		if(deviceId == SimSettings.GENERIC_EDGE_DEVICE_ID){
			//Select VM on edge devices via Least Loaded algorithm!
			double selectedVmCapacity = 0; //start with min value
			for(int hostIndex=0; hostIndex<numberOfHost; hostIndex++){
				List<EdgeVM> vmArray = SimManager.getInstance().getEdgeServerManager().getVmList(hostIndex);
				for(int vmIndex=0; vmIndex<vmArray.size(); vmIndex++){
					double requiredCapacity = ((CpuUtilizationModel_Custom)task.getUtilizationModelCpu()).predictUtilization(vmArray.get(vmIndex).getVmType());
					double targetVmCapacity = (double)100 - vmArray.get(vmIndex).getCloudletScheduler().getTotalUtilizationOfCpu(CloudSim.clock());
					if(requiredCapacity <= targetVmCapacity && targetVmCapacity > selectedVmCapacity){
						selectedVM = vmArray.get(vmIndex);
						selectedVmCapacity = targetVmCapacity;
					}
				}
			}
		}
		else{
			SimLogger.printLine("Unknown device id! The simulation has been terminated.");
			System.exit(0);
		}
		
		return selectedVM;
	}

	@Override
	public void processEvent(SimEvent arg0) {
		// Nothing to do!
	}

	@Override
	public void shutdownEntity() {
		// Nothing to do!
	}

	@Override
	public void startEntity() {
		// Nothing to do!
	}

}

```

这里主要是改了一下样例3的边缘编排器。config.properties中定义了一种策略TEST_POLICIES，在编排器类中选择负载最小的虚拟机进行处理任务。

## 5.定义网络模型

网络模型抽象类NetworkdModel，构造函数包含移动设备数目和模拟场景两个参数。继承NetworkModel需要重写getUploadDelay获取上传延迟方法和getDownloadDelay获取下载延迟方法。也可以根据需要自定义获取延迟时间的方法，供其他模块调用。

#### MM1Queue

MM1Queue是示例的网络模型方法，主要使用了M/M/1模型，wlan和wan时延满足泊松分布。getUploadDelay和getDownDelay用于获取时延，源设备固定是移动设备，根据目标设备不同，时延计算方式也有所不同。

网络模型使用默认的。

## 6.定义移动设备管理器

移动设备管理器抽象类MobileDeviceManager，其中有三个抽象方法

- initialize()初始化方法
- getCpuUtilizationModel()用于获取CPU使用模型
- submitTask方法根据任务特征来创建任务，调用编排器，将任务与相应虚拟机绑定，再调用schedule来安排任务。

## 7.定义移动服务器管理器

移动服务器管理器是除了云和边缘云服务器之外的可以移动的服务器，也能进行一些计算任务，也叫做本地计算。不需要这个模块也可以，默认的就是不支持本地计算。如果要实现，是通过继承MobileServerManager类。

## 8.定义负载生成器

通过继承LoadGeneratorModel抽象类实现，需要实现的方法有：initializeModel()根据任务生成模型来填充任务列表，getTaskTypeOfDevice(int)根据设备id返回该设备使用的任务类型。一个任务的特征是由应用决定的，应用由applications.xml文件定义的。applications.xml定义应用特征，类SimSettings对xml进行解析，将其放进名为taskLookUpTable的二维数组中。负载生成器就可以利用这个二维数组获取应用的特征。

## 9.编译脚本

编译脚本复制一下样例的，再根据需要修改一下，修改有关文件路径就可以用了。还有需要定义的就是配置文件。

#### config.properties基本配置

```properties
#default config file
simulation_time=33
warm_up_period=3
vm_load_check_interval=0.1
location_check_interval=0.1
file_log_enabled=true
deep_file_log_enabled=false

#定义不同移动设备数，会遍历每种场景每种设备数下的情况
min_number_of_mobile_devices=200
max_number_of_mobile_devices=200
mobile_device_counter_size=200

#时延、带宽
wan_propagation_delay=0.1
lan_internal_delay=0.005
wlan_bandwidth=0
wan_bandwidth=0
gsm_bandwidth=0

#all the host on cloud runs on a single datacenter
number_of_host_on_cloud_datacenter=1 
number_of_vm_on_cloud_host=4
core_for_cloud_vm=4
mips_for_cloud_vm=100000
ram_for_cloud_vm=32000
storage_for_cloud_vm=1000000

#移动设备不进行计算，核心数定义为0
core_for_mobile_vm=0
mips_for_mobile_vm=0
ram_for_mobile_vm=0
storage_for_mobile_vm=0

#use ',' for multiple values 
orchestrator_policies=TEST_POLICIES

#use ',' for multiple values
simulation_scenarios=TEST_SCENARIO

#mean waiting time in seconds
attractiveness_L1_mean_waiting_time=480
attractiveness_L2_mean_waiting_time=300
attractiveness_L3_mean_waiting_time=120
```

这里有很多字段，这里介绍主要的几个：

`attractiveness_Lx_mean_waiting_time`x是1，2，3之类的数字，代表不同的吸引力等级的平均等待时间，一个数据中心的吸引力等级在edge_devices.xml中定义。

`simulation_scenarios`是模拟场景名称。

`orchestrator_policies`是编排策略，就是一些策略的名称。在边缘编排器类中，对每种编排策略实现相应的逻辑。

#### applications.xml

用于定义应用特征，其中有很多个字段代表了应用的各种特性。

```xml
<?xml version="1.0"?>
<applications>
	<application name="TEST_APP">
		<usage_percentage>30</usage_percentage>
		<prob_cloud_selection>0</prob_cloud_selection>
		<delay_sensitivity>0.5</delay_sensitivity>
		<max_delay_requirement>0.5</max_delay_requirement>
		<poisson_interarrival>3</poisson_interarrival>
		<active_period>3600</active_period>
		<idle_period>1</idle_period>
		<data_upload>20</data_upload>
		<data_download>20</data_download>
		<task_length>3000</task_length>
		<required_core>1</required_core>
		<vm_utilization_on_edge>6</vm_utilization_on_edge>
		<vm_utilization_on_cloud>1.2</vm_utilization_on_cloud>
		<vm_utilization_on_mobile>0</vm_utilization_on_mobile>
	</application>
</applications>
```

在SimSettings类，即解析xml文件的类中有一段注释是对这些字段含义进行说明，当然也可以通过字面来了解它们的意思。

| 字段                     | 含义(单位)                |
| ------------------------ | ------------------------- |
| usage_percentage         | 使用比例(%)               |
| prob_cloud_selection     | 选择云的概率(%)           |
| poisson_interarrival     | 间隔的泊松平均(sec)       |
| active_period            | 有效阶段(sec)             |
| idle_period              | 空闲阶段(sec)             |
| data_upload              | 平均上传数据(KB)          |
| data_download            | 平均下载数据(KB)          |
| task_length              | 平均任务大小(MI)          |
| required_core            | 所需的核心数目            |
| vm_utilization_on_edge   | 边缘上的虚拟机利用率(%)   |
| vm_utilization_on_cloud  | 云上的虚拟机利用率(%)     |
| vm_utilization_on_mobile | 移动设备的虚拟机利用率(%) |
| delay_sensitivity        | 延迟敏感（范围0-1）       |
| max_delay_requirement    | 最大时延需求(sec)         |

#### edge_devices.xml边缘设备配置

边缘设备配置的内容如下，可以有多个数据中心，每个数据中心有自己的坐标和多个主机，每台主机有多个虚拟机。

```xml
<?xml version="1.0"?>
<edge_devices>
	<datacenter arch="x86" os="Linux" vmm="Xen">
		<costPerBw>0.1</costPerBw>
		<costPerSec>3.0</costPerSec>
		<costPerMem>0.05</costPerMem>
		<costPerStorage>0.1</costPerStorage>
		<location>
			<x_pos>1</x_pos>
			<y_pos>1</y_pos>
			<wlan_id>0</wlan_id>
			<attractiveness>0</attractiveness>
		</location>
		<hosts>
			<host>
				<core>1</core>
				<mips>10000</mips>
				<ram>2000</ram>
				<storage>50000</storage>
				<VMs>
					<VM vmm="Xen">
						<core>1</core>
						<mips>10000</mips>
						<ram>2000</ram>
						<storage>50000</storage>
					</VM>
				</VMs>
			</host>
		</hosts>
	</datacenter>
    <datacenter arch="x86" os="Linux" vmm="Xen">
		<costPerBw>0.1</costPerBw>
		<costPerSec>3.0</costPerSec>
		<costPerMem>0.05</costPerMem>
		<costPerStorage>0.1</costPerStorage>
		<location>
			<x_pos>1</x_pos>
			<y_pos>1</y_pos>
			<wlan_id>0</wlan_id>
			<attractiveness>1</attractiveness>
		</location>
		<hosts>
			<host>
				<core>1</core>
				<mips>10000</mips>
				<ram>2000</ram>
				<storage>50000</storage>
				<VMs>
					<VM vmm="Xen">
						<core>1</core>
						<mips>10000</mips>
						<ram>2000</ram>
						<storage>50000</storage>
					</VM>
				</VMs>
			</host>
		</hosts>
	</datacenter>
</edge_devices>
```

