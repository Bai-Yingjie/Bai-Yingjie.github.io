- [pktgen lua脚本](#pktgen-lua脚本)

# pktgen lua脚本
```lua
package.path = package.path ..";?.lua;test/?.lua;app/?.lua;../?.lua"


-- 加载Pktgen.lua, 该文件位于pktgen根目录下, 提供一些辅助函数
require "Pktgen";


function number2ip(number)
    -- string.format和printf挺像
    local str=string.format("%08x", number);
    -- 分组匹配, 连续赋值很赞
    local s4,s3,s2,s1 = str:match("(%x%x)(%x%x)(%x%x)(%x%x)");
    -- ..是字符串连接
    return string.format("%d.%d.%d.%d", "0x"..s4,"0x"..s3,"0x"..s2,"0x"..s1);
end


function number2mac(number)
    local str=string.format("%012x", number);
    local s6,s5,s4,s3,s2,s1 = str:match("(%x%x)(%x%x)(%x%x)(%x%x)(%x%x)(%x%x)");
    return string.format("%s:%s:%s:%s:%s:%s", s6,s5,s4,s3,s2,s1);
end


function setFlow(port, pkt_size, nb_L2, nb_L3, nb_L4)
    pktgen.range.ip_proto(port, "udp");


    pktgen.range.pkt_size(port, "start", pkt_size);
    pktgen.range.pkt_size(port, "inc", 0);
    pktgen.range.pkt_size(port, "min", pkt_size);
    pktgen.range.pkt_size(port, "max", pkt_size);


    pktgen.range.dst_mac(port, "inc", "00:00:00:00:00:00");


    pktgen.range.src_mac(port, "start", "00:00:00:00:00:01");
    pktgen.range.src_mac(port, "inc", "00:00:00:00:00:01");
    pktgen.range.src_mac(port, "min", "00:00:00:00:00:01");
    pktgen.range.src_mac(port, "max", number2mac(nb_L2));


    pktgen.range.dst_ip(port, "start", "0.0.0.1");
    pktgen.range.dst_ip(port, "inc", "0.0.0.1");
    pktgen.range.dst_ip(port, "min", "0.0.0.1");
    pktgen.range.dst_ip(port, "max", number2ip(nb_L3));


    pktgen.range.src_ip(port, "inc", "0.0.0.0");


    pktgen.range.dst_port(port, "inc", 0);


    pktgen.range.src_port(port, "start", 1);
    pktgen.range.src_port(port, "inc", 1);
    pktgen.range.src_port(port, "min", 1);
    pktgen.range.src_port(port, "max", nb_L4);

    pktgen.set_range(port, "on");
end


-- run 64B signle flow 10s or 100000000 packets, full rate, log performance data to file "whatever_64B_1L2-1L3-1L4_flow.data"
-- runFlowTest("whatever", 0, 0, 100, 64, 1, 1, 1, 10, 100000000)


-- run from pktgen shell
-- Pktgen:/> load /home/bai/repo/save/pktgen/range-flow.lua
-- Pktgen:/> lua 'runFlowTest("single", 0, 0, 100, 64, 1, 1, 1, 10, 0)'
-- Pktgen:/> lua 'runFlowTest("single", 0, 0, 100, 64, 1, 1, 1, 0, 100000000)'
-- Pktgen:/> lua 'runFlowTest("multi", 0, 0, 100, 64, 2000, 128, 128, 10, 0)'


function runFlowTest(mark, sendport, recvport, rate, pkt_size, nb_L2, nb_L3, nb_L4, duration, nb_pkts)  
    sendport = tostring(sendport);
    recvport = tostring(recvport);
    local datafile = tostring(mark) .. "_" .. pkt_size .. "B_" .. nb_L2 .. "L2-" .. nb_L3 .. "L3-" .. nb_L4 .. "L4_flow.data"
    local file = io.open(datafile, "w");


    pktgen.stop("all");
    pktgen.clr();
    pktgen.set(sendport, "rate", rate);
    pktgen.set(sendport, "count", nb_pkts);
    pktgen.latency("all", "enable");


    setFlow(sendport, pkt_size, nb_L2, nb_L3, nb_L4);


    pktgen.start(sendport);
    print("Running flow test and collecting data to " .. datafile);
    -- local如果在下面的repeat语句块里, 就只在该语句块内可见;
    local rateTx, rateRx;
    -- lua的table就是shell的关系数组, python的map

    local ppsTxTable = {};
    local ppsRxTable = {};
    local i = 0;
    repeat
    pktgen.delay(1 * 1000);
    rateTx = pktgen.portStats(sendport, "rate")[tonumber(sendport)];
    rateRx = pktgen.portStats(recvport, "rate")[tonumber(recvport)];
    i = i + 1;
    -- lua从1开始计数
    ppsTxTable[i] = rateTx.pkts_tx;
    ppsRxTable[i] = rateRx.pkts_rx;
    until(i == duration or rateTx.pkts_tx == 0)

    pktgen.stop(sendport);

    pktgen.delay(1 * 1000);
    -- i//2是"地板除法", lua数字都是浮点, 所以使用"地板除法", 只保留整数部分. 
    local ppsTx = ppsTxTable[i//2];
    local ppsRx = ppsRxTable[i//2];

    local statTx = pktgen.portStats(sendport, "port")[tonumber(sendport)];
    local statRx = pktgen.portStats(recvport, "port")[tonumber(recvport)];
    local num_tx = statTx.opackets;
    local num_rx = statRx.ipackets;
    local num_dropped = num_tx - num_rx;

    local statPkt = pktgen.pktStats("all")[tonumber(recvport)];
    -- file:write是个语法糖
    file:write("Tx pps: " .. ppsTx .. "\n");
    file:write("Rx pps: " .. ppsRx .. "\n");
    file:write("Tx pkts: " .. num_tx .. "\n");
    file:write("Rx pkts: " .. num_rx .. "\n");
    file:write("Dropped pkts: " .. num_dropped .. "\n");
    file:write("Min avg latency(usec): " .. statPkt.min_latency .. "\n");
    file:write("Max avg latency(usec): " .. statPkt.max_latency .. "\n");

    file:close();
    pktgen.set("all", "rate", 100);
    pktgen.set("all", "count", 0);
    pktgen.set_range("all", "off");
    pktgen.latency("all", "disable");
end
```
