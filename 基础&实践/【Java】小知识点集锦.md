
1. SpringMVC接受参数（int,Integer区别）
```	
       /**
	 * http://10.200.110.134:9308/api/test?a=1 OK
	 * http://10.200.110.134:9308/api/test 错误400
	 * */
	@GetMapping(value = "/test")
	public void test(@RequestParam int a ) {
		System.out.println("a" + a);
	}
	
	/**
	 * http://10.200.110.134:9308/api/testInteger?a=1    OK
	 * http://10.200.110.134:9308/api/testInteger   错误400
	 * */
	@RequestMapping(value = "/testInteger")
	public void testInteger(@RequestParam Integer a ) {
		System.out.println("a" + a);
	}
	
	/**
	 * http://10.200.110.134:9308/api/testInteger2?a=1    OK
	 * http://10.200.110.134:9308/api/testInteger2   OK
	 * */
	@RequestMapping(value = "/testInteger2")
	public void testInteger2(Integer a ) {
		System.out.println("a" + a);
	}	

```

2. SpringMVC接收String参数时equals、== 比较结果不一致原因
```
	UserResp user = new UserResp();
	user.setLoginType(new String("00"));//SpringMVC接受参数时会采用new String方式赋值，故Integer类似，比较时建议使用equals而非==
	System.out.println(user.getLoginType()==LoginTypeEnum._MERCHANT.getLoginType());
	System.out.println(user.getLoginType().equals(LoginTypeEnum._MERCHANT.getLoginType()));
	System.out.println("-----------------------------------------------------------------");
		
	UserResp user4 = new UserResp();
	String type = new String("00");
	user4.setLoginType(type.intern());
	System.out.println(user4.getLoginType()==LoginTypeEnum._MERCHANT.getLoginType());
	System.out.println(user4.getLoginType().equals(LoginTypeEnum._MERCHANT.getLoginType()));
	System.out.println("-----------------------------------------------------------------");

	UserResp user1 = new UserResp();
	user1.setLoginType("00");
	System.out.println(user1.getLoginType()==LoginTypeEnum._MERCHANT.getLoginType());
	System.out.println(user1.getLoginType().equals(LoginTypeEnum._MERCHANT.getLoginType()));
	System.out.println("-----------------------------------------------------------------");

	UserResp user2 = new UserResp();
	user2.setLoginType("0"+ "0");
	System.out.println(user2.getLoginType()==LoginTypeEnum._MERCHANT.getLoginType());
	System.out.println(user2.getLoginType().equals(LoginTypeEnum._MERCHANT.getLoginType()));
	System.out.println("-----------------------------------------------------------------");

	UserResp user3 = new UserResp();
	user3.setLoginType(String.valueOf("00"));
	System.out.println(user3.getLoginType()==LoginTypeEnum._MERCHANT.getLoginType());
	System.out.println(user3.getLoginType().equals(LoginTypeEnum._MERCHANT.getLoginType()));
	/*
	* false true -----------------------------------------------------------------
	* true true -----------------------------------------------------------------
	* true true -----------------------------------------------------------------
	* true true -----------------------------------------------------------------
	* true true
	*/

```

		 
		 


