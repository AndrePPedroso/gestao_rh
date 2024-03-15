# gestão RH

`python3 -m venv .venv`
`source .venv/bin/activate`
`pip install django`
`django-admin startproject gestao_rh .`  
`django-admin startapp departamentos`  
`django-admin startapp funcionarios`  
`django-admin startapp empresas` 
`django-admin startapp registro_de_hora_extra`  


Iniciamos um projeto de gestao de rh


**em setting.py adicionar**
``` python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_filters",
    "rest_framework",
    "..todos os apps",]
```

Nosso funcionario ira ter permissões como um funcionario do django, logo na model dele iremos importar o user .
cada funcionario fara parte de um departamento, uma empresa. desse modo podemos filtrar o que cada funcionario pode utilizar
``` python
class Funcionario(models.Model):
    nome = models.CharField(max_length=100)
    user = models.OneToOneField(User, on_delete=models.PROTECT)
    departamentos = models.ManyToManyField(Departamento)
    empresa = models.ForeignKey(
        Empresa, on_delete=models.PROTECT, null=True, blank=True)
    imagem = models.ImageField()
    de_ferias = models.BooleanField(default=False)
```
Nosso domentos terão uma foreign key para funcionario pois cada documento possui um funcionario.

``` python
class Documento(models.Model):
    descricao = models.CharField(max_length=100)
    pertence = models.ForeignKey(Funcionario, on_delete=models.PROTECT)
    arquivo = models.FileField(upload_to='documentos')
```

Em hora extra temos que cada funcionario pode ter mais que uma hora extra. 

``` python
class RegistroHoraExtra(models.Model):
    motivo = models.CharField(max_length=100)
    funcionario = models.ForeignKey(Funcionario, on_delete=models.PROTECT)
    horas = models.DecimalField(max_digits=5, decimal_places=2)
    utilizada = models.BooleanField(default=False)
```
Após organizou toda a questão do template . 

Quando o funcionario cria uma empresa, precisamos que ele seja vinculada a mesma logo: 

**em empresas.views.py adicionar**
``` python
class EmpresaCreate(CreateView):
    model = Empresa
    fields = ['nome']

    def form_valid(self, form):
        obj = form.save()
        funcionario = self.request.user.funcionario
        funcionario.empresa = obj
        funcionario.save()
        return HttpResponse('Ok')
```

lista de funcionarios da mesma empresa somente  

**em funcionarios.views.py adicionar**

``` python
  def get_queryset(self):
        empresa_logada = self.request.user.funcionario.empresa
        return Funcionario.objects.filter(empresa=empresa_logada)
```
**em funcionarios.model.py adicionar**

vamos criar uma property que vira um field de funcionario, para calcular todas as horas extras.
``` python
    @property
    def total_horas_extra(self):
        total = self.registrohoraextra_set.filter(
            utilizada=False).aggregate(
            Sum('horas'))['horas__sum']
        return total or 0
```

**para fazer com que quando damos get em um funcionario temos uma lista de suas horas extras** 
em serializer
``` python
class FuncionarioSerializer(serializers.ModelSerializer):
    registrohoraextra_set = RegistroHoraExtraSerializer(many=True)

    class Meta:
        model = Funcionario
        fields = (
            'id', 'nome', 'departamentos', 'empresa', 'user', 'imagem',
            'total_horas_extra', 'registrohoraextra_set')

```
``` python
 class HoraExtraList(ListView):
    model = RegistroHoraExtra

    def get_queryset(self):
        empresa_logada = self.request.user.funcionario.empresa
        return RegistroHoraExtra.objects.filter(
            funcionario__empresa=empresa_logada) 
```
funcionario__empresa chega no field empresa do funcionario que esta no registrohoraextra
