## OgnlUtil
* setProperties(Map<String, ?> ***props***, ...)
```Java
        for (Map.Entry<String, ?> entry : props.entrySet()) {
            String expression = entry.getKey();
            internalSetProperty(expression, entry.getValue(), o, context, throwPropertyExceptions);
        }
```
internalSetProperty会调用setValue，执行expression
* setProperty(String ***name***, ...)
```Java
        Ognl.setRoot(context, o);
        internalSetProperty(name, value, o, context, throwPropertyExceptions);
        Ognl.setRoot(context, oldRoot);
```
* setValue(final String ***name***, ...) throws OgnlException {
```Java
        compileAndExecute(name, context, new OgnlTask<Void>() {
            public Void execute(Object tree) throws OgnlException {
                if (isEvalExpression(tree, context)) {
                    throw new OgnlException("Eval expression/chained expressions cannot be used as parameter name");
                }
                if (isArithmeticExpression(tree, context)) {
                    throw new OgnlException("Arithmetic expressions cannot be used as parameter name");
                }
                Ognl.setValue(tree, context, root, value);
                return null;
            }
        });
```
对Ognl.setValue的封装，compileAndExecute函数会先分析name，生成对象tree，然后调用OgnlTask.execute(Object tree)函数，在里面再调用Ognl.setValue。
<br>
出了这么多漏洞，struts2也加了安全检查isEvalExpression和isArithmeticExpression，大佬们都有办法绕过吧
* getValue(final String ***name***, ...)
```Java
        return compileAndExecute(name, context, new OgnlTask<Object>() {
            public Object execute(Object tree) throws OgnlException {
                return Ognl.getValue(tree, context, root);
            }
        });
```
同样是对Ognl.getValue的封装
* callMethod(final String ***name***, ...)
```Java
        return compileAndExecuteMethod(name, context, new OgnlTask<Object>() {
            public Object execute(Object tree) throws OgnlException {
                return Ognl.getValue(tree, context, root);
            }
        });
```
继续封装Ognl.getValue
<br>
注意，对Ognl.getValue的封装都是没有isEvalExpression和isArithmeticExpression的检查哦，不用找绕过方法了
* copy(final Object from, ...)
```Java
        try {
            fromPds = getPropertyDescriptors(from);
            toPds = getPropertyDescriptors(to);
        }
        ...
        for (PropertyDescriptor fromPd : fromPds) {
            if (fromPd.getReadMethod() != null) {
        ...
                            compileAndExecute(fromPd.getName(), context, new OgnlTask<Object>() {
                                public Void execute(Object expr) throws OgnlException {
                                    Object value = Ognl.getValue(expr, contextFrom, from);
                                    Ognl.setValue(expr, contextTo, to, value);
                                    return null;
                                }
                            }
```
源对象的属性名被解析ognl了
* getBeanMap(final Object source)
```Java
        PropertyDescriptor[] propertyDescriptors = getPropertyDescriptors(source);
        for (PropertyDescriptor propertyDescriptor : propertyDescriptors) {
            final String propertyName = propertyDescriptor.getDisplayName();
            Method readMethod = propertyDescriptor.getReadMethod();
            if (readMethod != null) {
                final Object value = compileAndExecute(propertyName, null, new OgnlTask<Object>() {
                    public Object execute(Object expr) throws OgnlException {
                        return Ognl.getValue(expr, sourceMap, source);
                    }
                });
            }
        }
```
source对象的属性名也被解析ognl了
