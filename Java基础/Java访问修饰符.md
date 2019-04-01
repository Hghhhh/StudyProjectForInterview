public：所有类都可以访问到

private：只在类内部可以访问到

protected：包内的类和包外继承该类的子类可以访问到

default：包内的类才可以访问到



注意：protected的包外子类可以访问到指的是，继承的时候可以继承该方法，可以通过super访问到protected方法，不是只new一个父类来访问它的protected方法，这样是访问不到的。