# 9. 统计 OSD 上 PG 的数量

----------

我们可以通过一个 Python 脚本，统计出每个 OSD 上分布了多少个 PG ，以此判断集群的数据分布是否均衡。

    #!/usr/bin/env  python
	import sys 
	import os
	import json
	cmd = '''
	ceph pg dump | awk ' /^pg_stat/ { col=1; while($col!="up") {col++}; col++ } /^[0-9a-f]+\.[0-9a-f]+/ {print $1,$col}'
	'''
	body = os.popen(cmd).read()
	SUM = {}
	for line in  body.split('\n'):
   		if not line.strip():
        	continue
   		SUM[line.split()[0]] = json.loads(line.split()[1])
	pool = set()
	for  key in  SUM:
  		pool.add(key.split('.')[0])
	mapping = {}
	for number in pool:
  		for k,v in SUM.items():
    		if k.split('.')[0] == number:
       			if number in mapping:
           			mapping[number] += v
       			else:
           			mapping[number] = v
	MSG = """%(pool)-6s: %(pools)s | SUM
	%(line)s
	%(dy)s
	%(line)s
	%(sun)-6s: %(end)s |"""
	pools = " ".join(['%(a)-6s' % {"a": x} for x in sorted(list(mapping))])
	line = len(pools) + 20
	MA = {}
	OSD = []
	for p in mapping:
    	osd = sorted(list(set(mapping[p])))
    	OSD += osd
    	count = sum([mapping[p].count(x) for x in osd])
    	osds = {}
    	for x in osd:
        	osds[x] = mapping[p].count(x)
    	MA[p] = {"osd": osds, "count": count}
	MA = sorted(MA.items(), key=lambda x:x[0])
	OSD = sorted(list(set(OSD)))
	DY = ""
	for osd in OSD:
    	count = sum([x[1]["osd"].get(osd,0) for x in MA])
    	w = ["%(x)-6s" % {"x": x[1]["osd"].get(osd,0)} for x in MA]
    	#print w
    	w.append("| %(x)-6s" % {"x": count})
    	DY += 'osd.%(osd)-3s %(osds)s\n' % {"osd": osd, "osds": " ".join(w)}
	SUM = " ".join(["%(x)-6s" % {"x": x[1]["count"]} for x in MA])
	msg = {"pool": "pool", "pools": pools, "line": "-" * line, "dy": DY, "end": SUM, "sun": "SUM"}
	print MSG % msg

执行效果如下：

	root@OPS-ceph1:~# ./cal_pg_per_osd.py 
	dumped all in format plain
	pool  : 0      1      2      3      4      5      7      | SUM
	--------------------------------------------------------------------
	osd.0   2      9      11     10     5      6      1      | 44    
	osd.1   1      7      8      10     8      9      3      | 46    
	osd.2   2      11     7      6      6      8      4      | 44    
	osd.3   1      11     7      9      7      4      1      | 40    
	osd.4   2      12     12     12     11     13     0      | 62    
	osd.5   2      10     10     5      9      11     0      | 47    
	osd.6   3      56     47     49     38     43     16     | 252   
	osd.7   6      36     48     45     55     42     10     | 242   
	osd.8   4      41     42     37     35     49     15     | 223   
	osd.9   6      42     52     41     55     36     12     | 244   
	osd.10  10     36     47     51     39     43     15     | 241   
	osd.11  6      56     47     44     41     46     12     | 252   
	osd.12  6      40     45     45     51     46     11     | 244   
	osd.13  6      42     40     56     46     44     9      | 243   
	osd.14  5      44     41     49     48     52     7      | 246   
	osd.15  4      50     42     38     49     38     12     | 233   
	osd.16  3      9      5      11     8      8      2      | 46    
	osd.17  2      10     13     4      7      10     2      | 48    
	osd.18  0      12     10     10     10     9      6      | 57    
	osd.19  0      13     8      4      9      15     2      | 51    
	osd.20  1      11     7      9      9      15     2      | 54    
	osd.21  0      7      12     10     6      12     1      | 48    
	osd.22  4      39     45     30     43     46     12     | 219   
	osd.23  6      44     38     44     38     41     6      | 217   
	osd.24  7      37     50     49     49     36     8      | 236   
	osd.25  10     28     38     44     40     32     11     | 203   
	osd.26  1      44     33     47     47     37     13     | 222   
	osd.27  3      47     38     42     44     55     14     | 243   
	osd.28  4      47     47     34     33     48     11     | 224   
	osd.29  3      40     32     34     40     51     10     | 210   
	osd.30  8      39     46     47     55     35     10     | 240   
	osd.31  5      48     48     45     41     36     11     | 234   
	osd.32  5      46     48     53     42     48     7      | 249   
	
	--------------------------------------------------------------------
	SUM   : 128    1024   1024   1024   1024   1024   256    |