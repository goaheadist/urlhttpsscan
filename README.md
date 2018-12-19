# urlhttpsscan
readme
小业务部门域名HTTPS探测器
【使用场景】IT运营人员修改并运行 demo.sh 输入是主域名，修改demo.sh文件中第一个命令行一个域名
输出是output_file.txt文件，其中罗列了所有子域名，并判断该网站是否支持Https访问。
false表示无法进行HTTPS访问。
【环境搭建】
1. 基础环境搭建：python2, node.js, JDK, odpsTunnelClient，
2. 运行代码准备：subDomainsBrute, ali_health_domains table,index.js, update.py; 后3个是自主代码
3. 展示平台准备：aliQuark，
【使用】
1. 进入 subDomainsBrute-master文件夹运行 demo.sh
2. 在d2可以query 表格看中间结果
3. 在quark可以看图形结果

========
##【环境搭建】
### 1. 	基础环境搭建：都使用官网最新版本；
	/*
	* odps tunnel installation:
	*/
	http://odps.alibaba-inc.com/official_downloads/odpscmd/?spm=a2c1f.8259796.3.7.2db296d5BXeUEe
  只需要配合JDK使用
	修改access_id, access_key,来源myprofile https://datastudio2.dw.alibaba-inc.com/
	
	修改odps_config.ini
	project_name=sectech_dev
	access_id=xx
	access_key=xx
	end_point=http://service-corp.odps.aliyun-inc.com/api
	/*
	* odps tunnel installation:
	*/ 
### 2. 运行代码准备：
	2.1 subDomainsBrute
	/*
	* run ljj
	*/
	1. install 
	python2 install gevent
	error: could not create '/System/Library/Frameworks/Python.framework/Versions/2.7/include/python2.7/greenlet': Operation not permitted
	
	solution:
	sudo pip install gevent --user
	
	2. run
	go to ljj folder and run 
	sudo python subDomainsBrute.py qq.com
	
	//必须写sudo，否则报错can not import gevent
	/*
	* run ljj
	*/
	/*
	* 修改输出格式，作为后面自研代码index.js的输入
	*/
	修改line190 
	from：
	self.outfile.write(cur_sub_domain.ljust(30) + '\t' + ips + '\n')
	to：
	self.outfile.write('"'+ cur_sub_domain +'",' + '\n')

	2.2 把tunnel 根目录“odps_clt_release_64”copy到和subDomainsBrute-master并列到位置
	这是为了自研 update.sh文件中 ../odps_clt_release_64/bin/odpscmd <<< $SQL
	
	2.3 在subDomainsBrute-master根目录下创建3个文件内容如下。
		运行 
		chmod 755 demo.sh
		chmod 755 update.sh
	2.4 在aliODPS上创建表格 ali_health_domains 内容如下
		https://datastudio2.dw.alibaba-inc.com/
		-- DROP TABLE IF EXISTS ali_health_domains;
		-- select * from ali_health_domains
		CREATE TABLE IF NOT EXISTS ali_health_domains 
		(
		    id BIGINT COMMENT '*',
		    domain STRING COMMENT '*',
		    https STRING  COMMENT 'na', -- https, http, or na -- 302，200，404
		    product_line STRING COMMENT '*',
		    expiry_date STRING COMMENT '*',
		    contactor STRING  COMMENT '*',
		    description STRING COMMENT '*'
		)
		COMMENT '*';
3. 展现平台准备
	数据集-加速配置不要
	报表 - 缓存不要 
	https://quark.alibaba-inc.com/manage/dataset/list.htm?spm=a2o1z.8190073.0.0.32771051dS34cP
``
/*
* demo.sh
*/

# 《小业务部门域名HTTPS探测器》
#1. 收集：从主域名爬取所有子域名，解决新业务用到的子域名被遗漏的问题。修改qq.cn，作为后面自研代码index.js的输入
sudo python subDomainsBrute.py qq.cn -o input_file.txt


#2. 检测：从域名判断该网站是否支持Https访问。解决域名安全访问问题。
node index.js input_file.txt output_file.txt

#3. 存储：把域名检测中间结果，以文件上传方式存储到ODPS后台数据表
./update.sh

#4. 展现：1）图文报表展现，以quark的方式。 2）钉钉机器人问答展现
# https://quark.alibaba-inc.com/dashboard/view/page.htm?spm=a2o1z.8190073.0.0.32771051dS34cP&id=250292

/*
* demo.sh end of file
*/
``
/*
* update.sh
*/
ofile="output_file.txt"

# https://help.aliyun.com/document_detail/27833.html
SQL="tunnel upload  $ofile sectech_dev.ali_health_domains;"

../odps_clt_release_64/bin/odpscmd <<< $SQL
/*
* update.sh end of file
*/	

/*
* index.js
*/	
const http = require('http');
const fs = require('fs');
const MAX_CONN = 100;
const REQ_OPT = {
	timeout: 30 * 1000,
	headers: {
		'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
		'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7,ja;q=0.6,la;q=0.5,af;q=0.4',
		'Accept-Encoding': 'gzip, deflate, br',
		'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36',
	},
};

let domains;
let nNum = 0;
let nErr = 0;
let nSafe = 0;
let nUnsafe = 0;
let results = '';

var ofile;
var timer;

function load(path) {
	let text = fs.readFileSync(path, 'UTF8');
	//text = text.substr(0, text.length - 1);
	text = text.substring(0, text.lastIndexOf(','));
	text ='['+text+']';
	let list = JSON.parse(text);
	return list
		.filter(v => !v.startsWith('*'))
		.reverse();

}
var total;
function task() {
	if (nNum++ === total) {
		console.log('done');
		// fs.
		update();
		clearInterval(timer);
		process.exit(0);
		return;
	}
	domain = domains.pop();
	if (!domain) {
		return;
	}

	let url = 'http://' + domain + '/';
	http.get(url, REQ_OPT, res => {
		let code = res.statusCode;
		let redir = res.headers['location'] || '';
		let isSafe = false;

		if (300 < code && code < 400 && redir.startsWith('https:')) {
			isSafe = true;
			nSafe++;
		} else {
			nUnsafe++;
		}
		// results += `${domain},${code},${redir},${isSafe},,,\n`;
		results += `,${domain},${isSafe},,,,\n`;
		task();
	})
	.on('error', err => {
		nErr++;
		// console.log('error:', domain);
		task();
	})
	.on('timeout', _ => {
		nErr++;
		console.log('timeout:', domain);
		task();
	});
}

function update() {
	console.log('scaned:', nNum, 'err:', nErr, 'safe:', nSafe, 'unsafe:', nUnsafe);
	if (results) {
		fs.appendFileSync(ofile, results);
		results = '';
	}
}


function main(args) {
	if (args.length < 2) {
		console.error('usage: node . input_file output_file');
		return;
	}
	ofile = args[1];
	fs.writeFileSync(ofile, '');


	domains = load(args[0]);
	total = domains.length;
	console.log('domain count:', domains.length);

	//for (let i = 0; i < MAX_CONN; i++) {
		task();
	//}
	timer = setInterval(update, 1000);
}

main(process.argv.slice(2));

/*
* index.js end of file
*/	




	
