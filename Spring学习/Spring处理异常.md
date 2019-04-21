通过@ExceptionHandler标签来实现，value只能写明处理的异常类型

	@ExceptionHandler(value={Exception.class})//value可以是数组
	 public String handlerException(){
	  return "redirect:SelectAll.do";
	}
