Piensa en una aplicación bastante... bastante sencilla que calcula la suma de las áreas y perímetros de un conjunto de rectángulos. Para esto, tienes una clase llamada `Rectangle` que se ve más o menos así:

```
public class Rectangle
{
    public double Sides { get; } = 4;
    public double Height { get; set; }
    public double Width { get; set; }

    public static double SumAreas(IEnumerable<Rectangle> rectangles)
    {
        //   
    }

    public static double SumPerimeters(IEnumerable<Rectangle> rectangles)
    {
        //
    }
}
```

Mientras que el programa principal se encarga de crear los `rectángulos` y llamar a los métodos respectivos para obtener las sumas:

```
var rectangulos = new[]
{
    new Rectangle {Width = 10, Height = 5},
    new Rectangle {Width = 4, Height = 6},
    new Rectangle {Width = 5, Height = 1},
    new Rectangle {Width = 8, Height = 9}
};

var sumaDeAreas = Rectangle.SumAreas(rectangulos);
var sumaDePerimetros = Rectangle.SumPerimeters(rectangulos);
```  

Todo parece perfecto, tu programa funciona de mil maravillas, pero está violando el principio de responsabilidad única.  

## Violación del SRP  
La violación ocurre al momento de declarar los métodos `SumAreas` y `SumPerimeters` dentro de la misma clase que `Rectangle` y es que a pesar de que están relacionadas con el rectángulo como tal, la sumatoria forma parte de nuestra lógica de la aplicación, no de la lógica que podría tener un rectángulo en la vida real.  

## Cumpliendo el SRP  
Para cumplir con el principio, quitamos la funcionalidad de sumatorias de la clase `Rectangle` e introducimos un par de clases encargadas de realizar las operaciones sobre el los conjuntos de rectángulos, su código es más o menos este:  

```
public class AreaOperations
{
    public static double Sum(IEnumerable<Rectangle> rectangles)
    {
        //
        

public class PerimeterOperations
{
    public static double Sum(IEnumerable<Rectangle> rectangles)
    {
        //
```  
Entonces así cada clase tiene una sola responsabilidad: una representa un rectángulo y las otras se encargan de hacer operaciones relacionadas con ellos.  

Ahora, supón que después de cierto tiempo, a la gente le gustó tanto tu programa que te piden que ahora tomes en cuenta también triángulos equiláteros y que tu programa permita sumar las áreas de triángulos con rectángulos.  

Entonces creas una nueva clase para representar los triángulos, y modificas los métodos para sumar áreas  para que estos acepten tanto rectángulos como triángulos y dentro de ellos verificas de qué tipo es cada objeto y realizas el cálculo apropiado: 

```
public class PerimeterOperations
{
    public double Sum(IEnumerable<object> shapes)
    {
        double perimeter = 0;
        foreach (var shape in shapes)
        {
            if (shape is Rectangle)
                perimeter += 2 * ((Rectangle)shape).Height + 2 * ((Rectangle)shape).Width;
            else if (shape is EquilateralTriangle)
                perimeter += ((EquilateralTriangle) shape).SideLength * 3;
        }
        return perimeter;
    }
}

public class AreaOperations
{
    public double Sum(IEnumerable<object> shapes)
    {
        double area = 0;
        foreach (var shape in shapes)
        {
            if(shape is Rectangle)
                area += ((Rectangle)shape).Height * ((Rectangle)shape).Width;
            else if(shape is EquilateralTriangle)
                area += Math.Sqrt(3) *  Math.Pow(((EquilateralTriangle)shape).SideLength,2) / 4;
        }
        return area;
    }
}
```  

Todo parece perfecto nuévamente, tu programa funciona de mil maravillas, pero está violando el principio de abierto/cerrado.  

## Violación del OCP
Probablemente ya tengas una idea de en qué parte del código se está violando este principio... pero si no: en las clases de operaciones, y es que tu programa está abierto a la extensión... pero no a la modificación. Ponte a pensar en qué va a pasar mañana que los círculos se pongan de moda. Tendrías que modificar el código de las operciones para que funcione con otra figura y así para cada figura que se le ocurra a los usuarios de tu programa.   

## Cumpliendo el OCP  
La solución a esta violación se dará mediante el uso de abstracciones (en este caso la interfaz `IGeometricShape`) a través de la cual indicaremos que nuestras figuras comparten propiedades y métodos (en este caso el área y el perímetro):

```
public interface IGeometricShape
{
    double Area();
    double Perimeter();
}
```

Y también tenemos que modificar las clases de operaciones para que acepten ahora objetos que cumplan con ese comportamiento, para así poder llamar a esos métodos, sin tener que preocuparse de qué tipo son "realmente" los objetos con los que está trabajando:  

```
public double Sum(IEnumerable<IGeometricShape> shapes)
{
    double area = 0;
    foreach (var shape in shapes)
        area += shape.Area();
    return area;
}

public double Sum(IEnumerable<IGeometricShape> shapes)
{
    double perimeter = 0;
    foreach (var shape in shapes)
        perimeter += shape.Perimeter();
    return perimeter;
}
```

De este modo para cuando en el futuro agreguémos nuevas figuras, únicamente tendremos que hacer que implementen ese comportamiento en común y no tendremos que modificar el código ya existente.  

Ahora la gente comienza a pedir que tu programa pueda calcular la información de figuras cuadradas, entonces tu creas una clase llamada `Square` que herede de la clase `Rectangle` que creaste hace unos cuantos pasos, después de todo, un cuadrado no es más que un rectángulo con una pequeña restricción. Y para cumplir con dicha restricción, cambiaste el comportamiento de sus propiedades `Height` y `Width`:  

```
public class Square : Rectangle
{
    private double _height;
    private double _width;

    public override double Height
    {
        get { return _height; }
        set
        {
            _height = value;
            _width = value;
        }
    }

    public override double Width
    {
        get { return _width; }
        set
        {
            _width = value;
            _height = value;
        }
    }
}
```

Ahora, inclusive ya hay quien está desarrollando aplicaciones con tu código. Todo parece perfecto, tu programa funciona de mil maravillas, pero está violando el principio de sustitución de Liskov.  

## Violación del LSP  
Supón que como parte del crecimiento de tu programa, decidiste comenzar a escribir pruebas unitarias, y escribiste una como la siguiente:

```
Rectangle rectangle = new Square();
rectangle.Width = 3;
rectangle.Height = 6;

double expected = 18;
double actual = rectangle.Area();

Assert.AreEqual(expected, actual);
```  

Hay algunas cosas raras... sin embargo el código compila y se ejecuta, sin embargo la prueba falla. Y es que de esto se trata el todo el principio de sustitución de Liskov: los subtipos de una clase deben comportarse siempre como esta. En otras palabras: deriva de una clase solo para agregarle capacidades, no para modificar las que ya cuenta.    

## Cumpliendo el LSP  
La solución es bastante simple: debemos hacer que `Square` no derive de `Rectangle`, y que en su lugar, implemente `IGeometrcShape`:

```
public class Square : IGeometricShape
```  

De ese modo nadie, ni tu mismo, podrá pensar que en este ámbito los cuadrados y los rectángulos están relacionados.  

Ahora sí, todo parece perfecto, tu programa funciona de mil maravillas, pero está violando el principio de segregación de la interfaz.  

## Violación del ISP  
Sin tener la menor intención nosotros introdujimos esta violación cuando cumplimos con el OCP, y es que el ISP nos indica que debemos separar las interfaces para que los componentes de software que trabajan con ellas tengan únicamente la información que de ellas necesitan y no más. Vamos a echarle un vistazo a la interfaz `IGeometricShape`:

```
public interface IGeometricShape
{
    double Area();
    double Perimeter();
}
```  

Y se usa tanto en el cálculo de suma de áreas y en el de perímetros. Pero, ¿por qué tendría que saber la clase encargada de sumar los perímetros, que los elementos con los que trabaja también poseen un área? 

## Cumpliendo el ISP  
El cumplir con este principio nos lleva a separar la interfaz `IGeometricShape` en dos: `IHasPerimeter` y `IHasArea`, para así pasarle únicamente la información necesaria a cada uno de los métodos dentro de nuestro programa:
```
public interface IHasArea
{
    double Area();
}

public interface IHasPerimeter
{
    double Perimeter();
}

public interface IGeometricShape : IHasArea, IHasPerimeter
{
}
```

Tu código comienza a crecer, fantástico, así que piensas que es mejor seguir organizándolo y mueves toda la lógica del cálculo de sumas a otra clase:

```
public class GreatCalculator
{
    public double TotalAreas { get; private set; }
    public double TotalPerimeters { get; private set; }

    public void Calculate()
    {
        var figuras = new IGeometricShape[]
        {
            new Rectangle {Width = 10, Height = 5},
            new EquilateralTriangle {SideLength = 5},
            new Rectangle {Width = 4, Height = 6},
//...
```

Y realizaste las modificaciones necesarias al método principal del programa:

```
private static void Main(string[] args)
{
    var calculator = new GreatCalculator();
    calculator.Calculate();
    Console.WriteLine($"Área total: {calculator.TotalAreas}\nPerímetro total:{calculator.TotalPerimeters}");
    Console.ReadKey();
}
```

Tu código está listo para ser expandido aún más. Todo parece perfecto, tu programa funciona de mil maravillas, pero está violando el principio de inversión de dependencia.  

## Violación del ISP  
Esta violación ocurre en la clase nueva que acabas de agregar, justo en el método `Calculate`, y es que este él mismo está creando las figuras con las que opera (en el arreglo `figuras`). ¿Qué va a pasar en el futuro cuando se quiera añadir otra figura? ¿o cuando se quiera cambiar el tamaño de algunas de las figuras ya existentes?

## Cumpliendo el DIP  
Para cumplir con este principio tenemos que remover la dependencia que la clase `GreatCalculator` tiene con el arreglo `figuras`, haciendo que el objeto que la llame sea el encargado de proveerle con las figuras con las que tiene que operar:

```
public void Calculate(IEnumerable<IGeometricShape> figuras)
{
```

De este modo se reduce su dependencia, y está preparado para operar con cualquier número de `IGeometricShape`s que deseemos.

Y listo, hemos cumplido con los principíos SOLID.