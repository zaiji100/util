#linux shell 部分
``` shell

#
# this is the default value of orignDir where contains a .war file to be processed
# and you can change this value by $1 (you should be sure that the .war file is really exists)
#
#orignDir='/Users/feng.congyue/Documents/feng/workspace/hotelsweb/target'
#orignFile='hotelsweb.war'
#
# this is the default value of baseDir where will process the .war file
# and you cat change this value by $2  (you should be sure that the path is really exists)
#
baseDir='/Users/feng.congyue/Desktop/package'
orignDir=$baseDir/trunk
orignFile='hotelsweb.war'

#this is the entrance of the shell
function init(){
  [ -z $2 ] || orignDir=$2
  [ -z $3 ] || baseDir=$3
  [ -z $4 ] || orginFile=$4
  #0.create the directory if not exists
  initBaseDir $baseDir $@ ;

  #1.checkout or update the source from svn
  updateSource ;

  #2.build the project
  cd $orignDir
  mvn install

  #3.copy and unzip the hotelsweb.war file
  copyAndUnzip $@ ;

  #4.remove the lib directory if does not need a whole package
  [ -z $1 ] && removeLib ;

  #5.resourve cache
  processFileName  ;

  #6.repackage and upload to server
  zipAndScp ;
}

function initBaseDir(){
  [ -e $1/data ] || mkdir $1/data
  [ -e $1/war ] || mkdir $1/war
  [ -e $orignDir ] || mkdir $orignDir
  [ -e $origndir/target ] || mkdir $orignDir/target
}

function updateSource(){
  [ -e $origndir/.svn ] && svn up
  [ -e $orignDir/.svn ] || svn co http://10.0.1.200/svn/hotels_webserver/trunk/ $orignDir
}

function copyAndUnzip(){
  cp -r $orignDir/target/$orignFile $baseDir/war/ ;
  cd $baseDir/data ;
  unzip $baseDir/war/$orignFile ;
  rm -rf $baseDir/war/$orignFile ;
}

function removeLib(){
  rm -rf $baseDir/data/WEB-INF/lib ;
}

function processFileName(){
  node $baseDir/bin/resolveCache.js
}

function zipAndScp(){
  cd $baseDir/data/ ;
  zip -r $baseDir/war/hotelsweb.war ./* ;
  rm -rf $baseDir/data/* ;
  scp -i ~/test_webserver.pem ~/Desktop/package/war/hotelsweb.war  ec2-user@test3.xiayizhan.mobi:/home/ec2-user/
}

init $@;

```


#nodejs 部分 --- 处理文件

``` javascript

var fs=require('fs')


/*****************  config start ***************/
var path=['/Users/feng.congyue/Desktop/package/temp/customer'];

classifyFiles(path,['js','css','html','jsp']);
var renameFiles = __temp.js.concat(__temp.css);
var htmlFiles = __temp.js.concat(__temp.html).concat(__temp.jsp);

/*****************  config end ***************/

/*****************  调用区 start ***************/


processAllFileName(path,htmlFiles,renameFiles);

/*****************  调用区 end ***************/

/*****************  方法区 start ***************/


/**
 * @function 递归扫描 path下的文件
 * 	1. 对 path 目录及其子目录下，文件名包含在 renameFiles 数组中的所有文件进行如下规则重命名
 * 			fileNamePreix + time + fileNameSuffix
 * 			@example 
 * 				原文件 fileName = index.js  time=1430882225708
 * 				改变后 fileName = index1430882225708.js
 *  2. 对 path 目录及其子目录下，文件名包含在 htmlFiles 数组中的所有文件的内容进行如下更改：
 * 			将文件中对 renameFiles 中文件的引用
 * 			更改名称为 renameFiles 重命名后的名字。
 * 
 * @param {String} path  要进行处理的根目录
 * @param {Array} htmlFiles  要进行处理的HTML文件名称
 * 		html | jsp etc.
 * @param {Array} renameFiles  要进行重命名的文件名称
 * 		js 文件 或者 css 文件
 * @param {TimeString} time	一个当前时间戳在文件末尾加上时间戳--清除缓存的一种手段
 */
function processAllFileName(path,htmlFiles,renameFiles){
	if(typeof __temp == 'undefined'){
		__temp = {};
	}
	if(typeof __temp.__now == 'undefined'){
		__temp.__now = new Date().getTime();
	}
	path.forEach(function(subPath){
		fs.stat(subPath,function(error,stat){
			if(error!=null){
				console.log('fs.stat error');
			}else{
				if(stat.isFile()){
					//如果是 html 文件
					if(new RegExp(htmlFiles.join('|')+'$').test(subPath)){//修改文件内容中 饮用的 js 文件名称
						fs.readFile(subPath,{encoding:'utf8'},function(err,data){
							if(err){
								console.log('\n 读取文件: '+subPath+' 的内容出现异常')
							}else{
								renameFiles.forEach(function(value){
									data=data.replace(new RegExp(value,'g'),
										value.substring(0,value.lastIndexOf('.')) + __temp.__now + value.substring(value.lastIndexOf('.')));
								});
								fs.writeFile(subPath,data,{encoding:'utf8'},function(err){
									if(err){
										console.log('\n写文件：'+subPath+"出现错误～");
									}else{
										//console.log('\n写文件：'+subPath+"成功！")
									}
								});
							}
						});
					}
					
					if(new RegExp(renameFiles.join('|')+'$').test(subPath)){//css || js 文件--增加文件后缀名称
						var newName=subPath.substring(0,subPath.lastIndexOf('.')) + __temp.__now + subPath.substring(subPath.lastIndexOf('.'));
						fs.rename(subPath,newName,function(err){
							if(err){
								console.log('\n修改文件 '+subPath+' 的名字时出现错误！');
							}else{
								//console.log('\n 修改文件：'+subPath+' 的名字为：\n'+newName);
							}
						});
					}
				}else{
					fs.readdir(subPath,function(error,children){
						children.forEach(function(value){
							 processAllFileName([subPath+'/'+value],htmlFiles,renameFiles);
						});
					});
				}
			}
		});
	});
}


/**
 * 
 * @param {Array} pathArray
 * 		代表一个或多个路径的一个字符串数组
 * 			e. ['/etc']
 * 		or
 * 			e. ['/etc/a','/ect/b']
 * @param {Array} suffixArray
 * 		代表文件后缀名的数组
 * 			e.['js']  --  找出所有以 .js 结尾的文件  如 {'js':['a.js', 'b.js']}
 * 			e.['js','css']	-- 找出所有以 .js 或者 .css 文件结尾的文件 如 {'js':['a.js','b.js'],'css':['a.css','b.css']}
 * 			e.[]  -- 把目录下的所有文件进行归类
 *  @return {Object}
 * 		分类后的文件如下格式
 * 			{
 * 				'js' : ['a.js', 'b.js'],
 * 				'jpg' : ['a.jpg', 'b.jpg'],
 * 				...
 * 			}
 */
function classifyFiles(pathArray,suffixArray){
	if(typeof __temp == 'undefined'){
		__temp = {};
		if(suffixArray.length==0){
			__temp.__reg = new RegExp('.');//找出所有的文件并且以后缀名成分类
		}else{
			__temp.__reg = new RegExp('^('+suffixArray.join('|')+')$');
		}
	} 
	pathArray.forEach(function(path){
		try{
			var statSync = fs.statSync(path); //用于判断文件类型【文件，目录，块设备..】
			if(statSync.isFile()){
				var preLength = path.lastIndexOf('/');
				var suffixIndex = path.lastIndexOf('.');
				var categoryName = path.substring(suffixIndex+1);
				var fileName = path.substring(preLength+1);
				
				if(__temp.__reg.test(categoryName)){
					if(typeof __temp[categoryName] == 'undefined'){
						__temp[categoryName] = [];
					} 
					if(__temp[categoryName] instanceof Array){
						__temp[categoryName].push(fileName);
					}
				}
			}else if(statSync.isDirectory()){
				try{
					var subPaths = fs.readdirSync(path);
					subPaths.forEach(function(subPath){
						classifyFiles([path+'/'+subPath],suffixArray);
					});
				}catch(e){
					console.log('读取目录"'+path+'"出错!错误信息如下:');
					console.log(e);
					console.log(e.stack);
				}
				
			}
		}catch(e){
			console.log(path + "是错误路径！详细错误信息如下：");
			console.log(e);
			console.log(e.stack);
		}
		
	});
	return __temp;
}

/*****************  方法区 end ***************/

```
