## 这是一个date widget

 1. 上滑下滑事件
 2. 渲染
  - 计算几行几列
  - 生成框架
  - 数值渲染

`````javascript
_calendar={
		/**
		 * 计算当前日期月份的天数
		 * @param Date 对象 或者代表月份的正整数【1,2..,11,12】 
		 */
		getDaysInOneMonth:function(date){
			var month,days=31;
			//参数数据校验
			if(date instanceof Date){
				month = date.getMonth()+1;
			}else if(typeof date == 'number'){
				if(date>12 || date <1){
					throw new TypeError("月份应该在[1,2...,12]集合中");
				}
				month = date;
			}else{
				throw new TypeError("需要一个日期类型或者数值类型的参数！");
			}
			//天数计算
			switch(month){
				case 4,6,9,11:{
					days=30;break;
				}case 2:{
					days=_calendar.getDaysOfSecondMonthByDate(date);
				}
			}
			return days;
		},
		/**
		 * 计算当前日期对应的年的二月天数
		 * 规则：四年一润，百年不润，四百年润
		 * @param Date 对象 或者代表年的正整数
		 */
		getDaysOfSecondMonthByDate:function(date){
			var year,days=28;
			//参数数据校验
			if(date instanceof Date){
				year = date.getYear()+1900;
			}else if(typeof date == 'number'){
				if(date>12 || date <1){
					throw new TypeError("代表年的数字应该是一个有意义的正整数");
				}
				year = date;
			}else{
				throw new TypeError("需要一个日期类型或者数值类型的参数！");
			}
			//天数计算
			if(year%400==0 || year%100!=0 && year%4==0){
				days=29;
			}
			return days;
		},
		/**
		 * 获取月初一是星期几
		 * 【0-6】-->【日-六】
		 * @param date
		 */
		getFirstDays:function(date){
			if(!(date instanceof Date)){
				throw new TypeError("需要一个日期类型的参数！");
			}
			date.setDate(1);
			return date.getDay();
		},
		/**
		 * 获取 date 所代表的月份
		 * 渲染行数
		 * @param date
		 * @return 渲染函数【用来生成日期table】
		 */
		getRows:function(date){
			if(!(date instanceof Date)){
				throw new TypeError("需要一个日期类型的参数！");
			}
			var day = this.getFirstDays(date);
			var totalDays = this.getDaysInOneMonth(date);
			var leftDays = totalDays - (7-day);
			var leftRows = Math.ceil(leftDays / 7);
			return 1+leftRows;
		}
}
``````
