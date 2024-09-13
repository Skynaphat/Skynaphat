#include "ns3/core-module.h"
#include "ns3/network-module.h"
#include "ns3/internet-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/applications-module.h"
#include "ns3/mobility-module.h"
#include "ns3/netanim-module.h"
#include "ns3/flow-monitor-module.h"

using namespace ns3;

NS_LOG_COMPONENT_DEFINE ("ThreeTierTopology");

int main (int argc, char *argv[])
{
    // Command line arguments
    CommandLine cmd;
    cmd.Parse (argc, argv);

    Time::SetResolution (Time::NS);
    LogComponentEnable ("ThreeTierTopology", LOG_LEVEL_INFO);

    // Create nodes for three tiers
    NodeContainer accessNodes;
    NodeContainer distributionNodes;
    NodeContainer coreNodes;

    accessNodes.Create (4); // Four access nodes
    distributionNodes.Create (2); // Two distribution nodes
    coreNodes.Create (1); // One core node

    // Point-to-Point Helpers
    PointToPointHelper p2p;
    p2p.SetDeviceAttribute ("DataRate", StringValue ("5Mbps"));
    p2p.SetChannelAttribute ("Delay", StringValue ("2ms"));

    // Install point-to-point links
    NetDeviceContainer accessDevices, distributionDevices, coreDevices;
    for (uint32_t i = 0; i < 4; ++i)
    {
        accessDevices.Add (p2p.Install (accessNodes.Get (i), distributionNodes.Get (0)));
    }

    distributionDevices.Add (p2p.Install (distributionNodes.Get (0), coreNodes.Get (0)));
    distributionDevices.Add (p2p.Install (distributionNodes.Get (1), coreNodes.Get (0)));

    // Install Internet stack
    InternetStackHelper stack;
    stack.Install (accessNodes);
    stack.Install (distributionNodes);
    stack.Install (coreNodes);

    // Assign IP addresses
    Ipv4AddressHelper address;
    address.SetBase ("10.1.1.0", "255.255.255.0");
    Ipv4InterfaceContainer accessInterfaces, distributionInterfaces, coreInterfaces;

    for (uint32_t i = 0; i < 4; ++i)
    {
        accessInterfaces.Add (address.Assign (accessDevices.Get (i)));
    }

    address.SetBase ("10.1.2.0", "255.255.255.0");
    distributionInterfaces.Add (address.Assign (distributionDevices.Get (0)));
    distributionInterfaces.Add (address.Assign (distributionDevices.Get (1)));

    address.SetBase ("10.1.3.0", "255.255.255.0");
    coreInterfaces.Add (address.Assign (coreDevices.Get (0)));

    // Set up applications
    UdpEchoServerHelper echoServer (9);
    ApplicationContainer serverApps = echoServer.Install (accessNodes.Get (0));
    serverApps.Start (Seconds (1.0));
    serverApps.Stop (Seconds (10.0));

    UdpEchoClientHelper echoClient (coreInterfaces.GetAddress (0), 9);
    echoClient.SetAttribute ("MaxPackets", UintegerValue (1));
    echoClient.SetAttribute ("Interval", TimeValue (Seconds (1.0)));
    echoClient.SetAttribute ("PacketSize", UintegerValue (1024));

    ApplicationContainer clientApps = echoClient.Install (accessNodes.Get (1));
    clientApps.Start (Seconds (2.0));
    clientApps.Stop (Seconds (10.0));

    // NetAnim setup
    AnimationInterface anim ("three_tier_topology.xml");
    anim.SetConstantPosition (accessNodes.Get (0), 10, 20);
    anim.SetConstantPosition (accessNodes.Get (1), 30, 20);
    anim.SetConstantPosition (distributionNodes.Get (0), 20, 40);
    anim.SetConstantPosition (coreNodes.Get (0), 20, 60);

    // Flow Monitor setup
    FlowMonitorHelper flowMonitor;
    Ptr<FlowMonitor> monitor = flowMonitor.InstallAll();

    Simulator::Stop (Seconds (11.0));
    Simulator::Run ();

    // Serialize Flow Monitor data
    monitor->SerializeToXmlFile ("flowmonitor.xml", true, true);

    Simulator::Destroy ();
    return 0;
}
