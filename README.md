# Análise dos dados de Geração Distribuída

Este projeto foi desenvolvido como parte das atividades do [Instituto Brasileiro de Defesa do Consumidor (Idec)](https://idec.org.br/), que está utilizando dados da [Agência Nacional de Energia Elétrica (ANEEL)](http://aneel.gov.br/) para analisar o perfil socioeconômico dos adotantes de Geração Distribuída no país.
  
A base de dados disponibilizada pela ANEEL ao Idec encontra-se [neste link](https://drive.google.com/drive/folders/1J_hsljUFAc-RBft0B4T0HV9JaxHi0Mb8?usp=sharing). Ela contém todas as ligações de GD realizadas no país até 24/nov/2020.

## Estruturando o banco de dados
Iremos estruturas os dados de GD em um banco de dados PostgreSQL para facilitar consultas e análises. Se você ainda não instalou o PostgreSQL e um visualizador/manipulador de dados, siga o passo 1 [deste tutorial](https://drive.google.com/drive/u/1/folders/1W2zOBsXvhnY_30wRv2TQ9UpIsQGRCVUG).
  
Caso já tenha instalado, use o código abaixo para criar a tabela PostgreSQL, importar os dados disponibilizados pela ANEEL e visualizar os dados. As etapas de visualização e checagem dos dados são importantes para garantir que a importação ocorreu como esperado.
  
Se você tiver dúvidas sobre o código abaixo, assita [esta aula](https://drive.google.com/drive/u/1/folders/1cicQE5kCjfuNlOSWdRcuz0LSacBN8Bp0) de introdução ao SQL.  
  

  
``` sql
-- Criando tabela ligacoes_gd, que conterá as infos das ligações de GD
create table ligacoes_gd (
	id_agente int,
	nm_agente varchar(150),
	sig_agente varchar(50),
	cd_uc varchar(100),
	cd_gd varchar(100),
	nm_titular_uc varchar(300),
	cpf_cnpj varchar(50),
	classe varchar(50),
	grupo varchar(50),
	modalidade varchar(100),
	qt_uc_credito int,
	nm_muni varchar(100),
	nm_regiao varchar(100),
	cd_cep varchar(50),
	dt_conexao date,
	tipo_gd varchar(50),
	combustivel_gd varchar(100),
	potencia_kw decimal(10,2)
);

-- Deleta a tabela, caso correções sejam necessárias 
drop table ligacoes_gd;

-- Copiando conteúdo do csv para a tabela ligacoes_gd
copy ligacoes_gd from 'csv_path' delimiter ';' csv header;

-- Visualizando os dados
select * from ligacoes_gd;

-- Checando dados: 334.364 ligações e 4.173.400,03 kW até 24/11/2020
select count(*) from ligacoes_gd;
select sum(potencia_kw) from ligacoes_gd;
```
  
## Limpeza dos dados
Utilize o código abaixo para realizar uma limpeza nos dados. Isso nos ajudará a trabalhar com eles no futuro.
  
  
```sql
-- Inserindo hífen no CEP para facilitar geocodificação
update ligacoes_gd set cd_cep = concat(substring(cd_cep, 1, 5), '-', substring(cd_cep, 6, 3));

-- Separando município e UF
alter table ligacoes_gd add column uf varchar(2);
alter table ligacoes_gd add column muni varchar(100);
update ligacoes_gd set uf = substring(nm_muni, length(nm_muni) - 1, 2);
update ligacoes_gd set muni = substring(nm_muni, 1, length(nm_muni) - 5);

```

## Analisando os dados gerais
Para analisar os dados, seguiremos [esta apresentação](https://docs.google.com/presentation/d/1WNsI5pIclzF1jfITanNuFwAOVEuzfs9VVfKRp5fMfEg/edit?usp=sharing), a partir do item 2. No código abaixo, estão as querys que permitem obter os dados necessários para gerar os gráficos da apresentação.
  
Nas querys, a diferenciação entre pessoas físicas e jurídicas é feita com a função `length(cpf_cnpj)`, já que CPF's sempre têm comprimento igual a 11 e CNPJ's sempre têm comprimento maior que 11.
  
```sql
-- Obtendo potência total instalada por pessoas físicas
-- Para pessoas jurídicas, altere para length(cpf_cnpj) > 11
select sum(potencia_kw)
from ligacoes_gd
where length(cpf_cnpj) = 11;

-- Obtendo total de ligações de pessoas físicas
select count(*)
from ligacoes_gd
where length(cpf_cnpj) = 11;

-- Obtendo total de titulares do tipo pessoa física
select count(distinct(nm_titular_uc ))
from ligacoes_gd lg 
where length(cpf_cnpj) = 11;

-- Obtendo potência total instalada por pessoas físicas por UF 
select sum(potencia_kw), uf
from ligacoes_gd
where length(cpf_cnpj) = 11
group by uf;
```
  
Para os gráficos que mostram a participação dos 10% maiores titulares na potência instalada, o código é um pouco mais complexo, pois foi necessário utilizar três funções: `potencia_total()`, `total_titulares()` e `potencia_10p_maiores_titulares()`. As funções estão definidas no código a seguir. Observe que, nos comentários, está indicado quais parâmetros devem ser passados e o que é retornado em cada uma delas.
  
```sql
-- Recebe um tipo ('cpf', 'cnpj' ou 'total') e um ano (2012 a 2020) e retorna a potência total instalada referente ao tipo e ao ano indicados
create or replace function potencia_total(varchar(4), integer) returns float as '
	declare
		tipo alias for $1;
		ano alias for $2;
		tipo_condicao varchar(50);
		resultado integer;
	begin
		select into resultado sum(potencia_instalada)
		from
			(select sum(potencia_kw) as potencia_instalada, nm_uniformizado, row_number() over (order by sum(potencia_kw) desc) as ranking
			from ligacoes_gd lg 
			where 
				case tipo
		          	when ''cpf'' then length(lg.cpf_cnpj) = 11
		          	when ''cnpj'' then length(lg.cpf_cnpj) > 11
		          	when ''total'' then length(lg.cpf_cnpj) >= 11
		      	end
			and (extract(year from dt_conexao)) <= ano
			group by nm_uniformizado) as titulares_rankeados;
		return resultado;			
	end;'
language 'plpgsql';

-- Recebe um tipo ('cpf', 'cnpj' ou 'total') e um ano (2012 a 2020) e retorna o total de titulares referente ao tipo e ao ano indicados
create or replace function total_titulares(varchar(4), integer) returns integer as '
	declare
		tipo alias for $1;
		ano alias for $2;
		resultado integer;
	begin
		select max(ranking) into resultado
		from 
			(select sum(potencia_kw) as potencia_instalada, nm_uniformizado, row_number() over (order by sum(potencia_kw) desc) as ranking
			from ligacoes_gd lg 
			where
				case tipo
		          	when ''cpf'' then length(lg.cpf_cnpj) = 11
		          	when ''cnpj'' then length(lg.cpf_cnpj) > 11
		          	when ''total'' then length(lg.cpf_cnpj) >= 11
		      	end
			and (extract(year from dt_conexao)) <= ano
			group by nm_uniformizado) as titulares_rankeados;
		return resultado;			
	end;'
language 'plpgsql';

-- Recebe um total de titulares (obtido com a função total_titulares(), um tipo ('cpf', 'cnpj' ou 'total') e um ano (2012 a 2020) e retorna a participação dos 10% maiores titulares na potência instalada
create or replace function potencia_10p_maiores_titulares(integer, varchar(4), integer) returns float as '
	declare
		total_titulares alias for $1;
		tipo alias for $2;
		ano alias for $3;
		resultado integer;
	begin
		select into resultado sum(potencia_instalada)
		from 
			(select sum(potencia_kw) as potencia_instalada, nm_uniformizado, row_number() over (order by sum(potencia_kw) desc) as ranking
			from ligacoes_gd lg 
			where
				case tipo
		          	when ''cpf'' then length(lg.cpf_cnpj) = 11
		          	when ''cnpj'' then length(lg.cpf_cnpj) > 11
		          	when ''total'' then length(lg.cpf_cnpj) >= 11
		      	end
			and (extract(year from dt_conexao)) <= ano
			group by nm_uniformizado) as titulares_rankeados
		where ranking <= round(0.1*total_titulares);
		return resultado;			
	end;'
language 'plpgsql';


-- Query que obtém os dados necessários para fazer o gráfico. Vá mudando o ano e o tipo de titularidade ('cpf', 'cnpj' ou 'total') para obter todos os dados.
select (select potencia_10p_maiores_titulares(total_titulares('cpf', 2020), 'cpf', 2020)) / (select potencia_total('cpf', 2020));

```

Os mapas foram elaborados utilizando [QGIS](https://qgis.org/pt_BR/site/) e o diagrama de sankey foi elaborado utilizando [este site](https://flourish.studio/visualisations/sankey-charts/).


## Analisando os dados de pessoas jurídicas (CNPJ's)
Para traçar o perfil socioeconômico dos adotantes do tipo pessoa jurídica, foi utilizada a API Minha Receita, que permite acessar gratuitamente dados da Receita Federal. De posse do número do CNPJ, foram obtidas informações como atividade principal, porte da empresa e seu capital social.

[Neste link](https://dev.to/camilacrdoso/obtendo-dados-de-cnpj-s-com-a-api-minha-receita-2hcd), encontra-se um tutorial sobre como utilizar a API Minha Receita e obter todos esses dados.

Uma vez que você tenha seguido o tutorial, você deve ter obtido uma tabela no seu banco de dados com todas as informações de cada CNPJ. Agora vamos combinar os dados de ligações de GD com esses dados de CNPJ's.

```sql
-- Combina os dados de ligações de GD (tabela ligacoes_gd) com os dados de CNPJ (tabela cnpj_data)
select * from ligacoes_gd lg 
left join cnpj_data cd 
on lg.cpf_cnpj = cd.cnpj
where length(lg.cpf_cnpj) > 11;

-- Potência instalada por grupo de atividade principal
select sum(potencia_kw),
	case when (atividade_principal_codigo::numeric < 400000) THEN 'AGRICULTURA, PECUÁRIA, PRODUÇÃO FLORESTAL, PESCA E AQÜICULTURA'
		 when (atividade_principal_codigo::numeric >= 500000 and atividade_principal_codigo::numeric < 1000000) then 'INDÚSTRIAS EXTRATIVAS'
		 when (atividade_principal_codigo::numeric >= 1000000 and atividade_principal_codigo::numeric < 3400000) then 'INDÚSTRIAS DE TRANSFORMAÇÃO'
		 when (atividade_principal_codigo::numeric >= 3500000 and atividade_principal_codigo::numeric < 3600000) then 'ELETRICIDADE E GÁS'
		 when (atividade_principal_codigo::numeric >= 3600000 and atividade_principal_codigo::numeric < 4000000) then 'ÁGUA, ESGOTO, ATIVIDADES DE GESTÃO DE RESÍDUOS E DESCONTAMINAÇÃO'
		 when (atividade_principal_codigo::numeric >= 4100000 and atividade_principal_codigo::numeric < 4400000) then 'CONSTRUÇÃO'
		 when (atividade_principal_codigo::numeric >= 4500000 and atividade_principal_codigo::numeric < 4800000) then 'COMÉRCIO; REPARAÇÃO DE VEÍCULOS AUTOMOTORES E MOTOCICLETAS'
		 when (atividade_principal_codigo::numeric >= 4900000 and atividade_principal_codigo::numeric < 5400000) then 'TRANSPORTE, ARMAZENAGEM E CORREIO'
		 when (atividade_principal_codigo::numeric >= 5500000 and atividade_principal_codigo::numeric < 5700000) then 'ALOJAMENTO E ALIMENTAÇÃO'
		 when (atividade_principal_codigo::numeric >= 5800000 and atividade_principal_codigo::numeric < 6400000) then 'INFORMAÇÃO E COMUNICAÇÃO'
		 when (atividade_principal_codigo::numeric >= 6400000 and atividade_principal_codigo::numeric < 6700000) then 'ATIVIDADES FINANCEIRAS, DE SEGUROS E SERVIÇOS RELACIONADOS'
		 when (atividade_principal_codigo::numeric >= 6800000 and atividade_principal_codigo::numeric < 6900000) then 'ATIVIDADES IMOBILIÁRIAS'
		 when (atividade_principal_codigo::numeric >= 6900000 and atividade_principal_codigo::numeric < 7600000) then 'ATIVIDADES PROFISSIONAIS, CIENTÍFICAS E TÉCNICAS'
		 when (atividade_principal_codigo::numeric >= 7700000 and atividade_principal_codigo::numeric < 8300000) then 'ATIVIDADES ADMINISTRATIVAS E SERVIÇOS COMPLEMENTARES'
		 when (atividade_principal_codigo::numeric >= 8400000 and atividade_principal_codigo::numeric < 8500000) then 'ADMINISTRAÇÃO PÚBLICA, DEFESA E SEGURIDADE SOCIAL'
		 when (atividade_principal_codigo::numeric >= 8500000 and atividade_principal_codigo::numeric < 8600000) then 'EDUCAÇÃO'
		 when (atividade_principal_codigo::numeric >= 8600000 and atividade_principal_codigo::numeric < 8900000) then 'SAÚDE HUMANA E SERVIÇOS SOCIAIS'
		 when (atividade_principal_codigo::numeric >= 9000000 and atividade_principal_codigo::numeric < 9400000) then 'ARTES, CULTURA, ESPORTE E RECREAÇÃO'
		 when (atividade_principal_codigo::numeric >= 9400000 and atividade_principal_codigo::numeric < 9700000) then 'OUTRAS ATIVIDADES DE SERVIÇOS'
		 when (atividade_principal_codigo::numeric >= 9700000 and atividade_principal_codigo::numeric < 9800000) then 'SERVIÇOS DOMÉSTICOS'
		 when (atividade_principal_codigo::numeric >= 9900000 and atividade_principal_codigo::numeric < 10000000) then 'SERVIÇOS DOMÉSTICOS'
		 when atividade_principal_codigo is null then 'Sem informação'
    end as atividades_principal_grupo
from 
    (select * from ligacoes_gd lg 
    left join cnpj_data cd 
    on lg.cpf_cnpj = cd.cnpj
    where length(lg.cpf_cnpj) > 11) as join_table
group by atividades_principal_grupo
order by sum(potencia_kw) desc;

-- Ligações por porte da empresa
select count(*), porte from 
	(select * from ligacoes_gd lg 
	left join cnpj_data cd 
	on lg.cpf_cnpj = cd.cnpj
	where length(lg.cpf_cnpj) > 11) as join_table
group by porte
order by count(*) desc;
    
-- Potência instalada por capital social
select sum(potencia_kw),
	case when capital_social >= 1000000000 THEN '> 1 bilhão'
		 when (capital_social >= 1000000 and capital_social < 1000000000) then 'entre 1 milhão e 1 bilhão'
		 when (capital_social >= 500000 and capital_social < 1000000) then 'entre 500 mil e 1 milhão'
		 when capital_social < 500000 then '< 500 mil'
		 when capital_social is null then 'Sem informação'
    end as faixa_capital_social
from 
	(select * from ligacoes_gd lg 
		left join cnpj_data cd 
		on lg.cpf_cnpj = cd.cnpj
		where length(lg.cpf_cnpj) > 11) as join_table
group by faixa_capital_social
order by sum(potencia_kw) desc;

```

Para obter as 20 empresas com maior potência instalada, primeiramente, teremos que adotar um nome uniformizado para as empresas. Isso é necessário porque, na base de dados da ANEEL, há algumas inconsistência em relação ao nome do titular e CNPJ, por exemplo:

|Titular (segundo ANEEL)        |     CNPJ (segundo ANEEL)    |   Razão social (segundo Receita Federal)   |
| ------------------------------|:---------------------------:| ------------------------------------------:|
| CENTRO EDUCACIONAL EDUCAR LTDA| 12194903000130              | EBES SISTEMAS DE ENERGIA SA                |

Há também casos em que a mesma empresa aparece com diferentes nomes de titular e diferentes CNPJ’s:

|Titular (segundo ANEEL)        |     CNPJ (segundo ANEEL)    |   Razão social (segundo Receita Federal)   |
| ------------------------------|:---------------------------:| ------------------------------------------:|
| CLARO S. A.                   | 40432544000147              | CLARO S.A.                                 |
| CLARO S A                     | 40432544062177              | CLARO S.A.                                 |
| TIM CELULAR SA                | 04206050007940              | TIM CELULAR S.A.                           |
| TIM SA                        | 02421421001606              | TIM S A                                    |

Assim, para identificar os maiores consumidores, não seria possível utilizar nem o nome do Titular (segundo ANEEL), nem o CNPJ, nem a razão social (segundo Receita Federal). Para resolver esse impasse, foi adotado um “nome uniformizado” que corresponde à razão social, com padronização para algumas empresas que possuem mais de uma razão social:

| Titular (segundo ANEEL) | CNPJ (segundo ANEEL) | Razão social (segundo Receita Federal) | Nome uniformizado |
| ------------------------|:--------------------:| --------------------------------------:|-------------------|
| CLARO S. A.             | 40432544000147       | CLARO S.A.                             | CLARO S.A.        |
| CLARO S A               | 4043254406217        | CLARO S.A.                             | CLARO S.A.        |
| TIM CELULAR SA          | 04206050007940       | TIM CELULAR S.A.                       | TIM CELULAR S.A.  |
| TIM SA                  | 02421421001606       | TIM S A                                | TIM CELULAR S.A.  |

O código abaixo define os nomes uniformizados adotados.

```sql
-- Cria um novo campo para armazenar os nomes uniformizados
alter table ligacoes_gd add column nm_uniformizado varchar(250);

-- Alimenta o novo campo, inserindo a razão social obtida na base da Receita Federal
update ligacoes_gd lg set nm_uniformizado = cd.razao_social
from cnpj_data as cd
where lg.cpf_cnpj = cd.cnpj;

-- Atualiza o campo para o caso da 'TIM CELULAR S.A.'
update ligacoes_gd set nm_uniformizado = 'TIM CELULAR S.A.' where nm_uniformizado = 'TIM S A';

-- Atualiza o campo para o caso das pessoas físicas, adotando o nome do titular segundo a ANEEL
update ligacoes_gd set nm_uniformizado = nm_titular_uc where length(cpf_cnpj) = 11;

```

Uma vez que o nome das empresas tenha sido atualizado, podemos obter as 20 empresas com maior potência instalada:

```sql
-- Obtém as 20 empresas com maior potência instalada
select
    sum(potencia_kw) as potencia_por_titular,
    count(*) as ligacoes_por_titular,
    nm_uniformizado,
    max(capital_social),
    mode() within group (order by atividade_principal_descricao)
from 
	(select * from ligacoes_gd lg 
	left join cnpj_data cd 
	on lg.cpf_cnpj = cd.cnpj
	where length(lg.cpf_cnpj) > 11) as join_table
group by nm_uniformizado
order by sum(potencia_kw) desc
limit 20;

```

## Analisando os dados de pessoas físicas

No caso das pessoas físicas, não é possível acessar os dados da Receita Federal. Por isso, para traçar o perfil socioeconômico dos adotantes do tipo pessoa física, utilizamos a localização das ligações de GD, que foi combinada com dados do Censo 2010 sobre distribuição espacial da renda média per capita nos municípios brasileiros.
  
Para realizar esta análise, tiramos proveito do fato de que a base de dados da ANEEL conta com o CEP das ligações de GD. Observe que temos apenas o CEP e não a latitude e longitude das ligações, entretanto, para fazer análises de dados espaciais, é necessário obter as coordenadas geográficas, então será necessário converter os CEP's nas latitudes e longitudes correspondentes. Esse processo de conversão de um CEP ou endereço em coordenadas geográficas é chamado de geocodificação.

### Geocodificando os CEP's das ligações de GD

[Neste link](https://github.com/camilacrdoso/geocode_with_google_api/blob/master/geocode_google_api.ipynb), encontra-se um tutorial sobre como geocodificar CEP's com a API de Geocodificação do Google.
  
Uma vez que você tenha seguido o tutorial, você deve ter obtido uma tabela com as latitudes e longitudes correspondentes aos CEP's. Utilize o código abaixo para combinar os dados de GD com a latitude e longitude de cada ligação e exportar as informações para um csv. Ao abrir esse csv com o [QGIS](https://qgis.org/pt_BR/site/), você conseguirá ver a localização geográfica das ligações de GD. Observe que o código abaixo já está filtrando apenas as ligações da classe Residencial e da modalidade Geração na própria UC.

```sql
copy
	(select * 
     from 
		(select
             sig_agente, 
             cd_uc, cd_gd,
             nm_titular_uc,
             cpf_cnpj,
             classe,
             grupo,
             modalidade,
             qt_uc_credito,
             lg.nm_muni,
             muni,
             uf,
             lg.cd_cep,
             dt_conexao,
             tipo_gd,
             combustivel_gd,
             potencia_kw, 
             lat,
             long
		from
             ligacoes_gd lg
		left join
             ceps_coord cp
		on
             lg.cd_cep = cp.cd_cep
      ) as tabela_completa
      where
         tabela_completa.nm_muni = 'São Paulo - SP'
         and lat is not null
         and classe = 'Residencial'
         and modalidade = 'Geracao na propria UC')
to 
    'C:\Users\Public\DadosGD_InfosUC_LatLong_RMSP.csv' delimiter ';' csv header;
```


  
### Acessando dados do Censo 2010  
Para acessar os dados do Censo 2010, siga os passos:
1. Acesse [este link](https://www.ibge.gov.br/estatisticas/sociais/trabalho/9662-censo-demografico-2010.html) no site do IBGE;
2. Navegue até Downloads > Censo 2010 > Resultados do Universo > Agregados por Setor Censitário;
3. Faça download da documentação e dos arquivos relativos aos estados que você deseja analisar;
4. Acesse [este link](https://www.ibge.gov.br/geociencias/organizacao-do-territorio/estrutura-territorial/26565-malhas-de-setores-censitarios-divisoes-intramunicipais.html) no site do IBGE;
5. Navegue até Downloads > Censo 2010 > Setores Censitários Shapefile;
6. Faça o download dos arquivos relativos aos estados que você deseja analisar.

Para gerar mapas de renda como os que estão na apresentação, você deve abrir o shapefile de setores censitário no QGIS e a tabela com os dados de renda (você irá descobrir qual é a tabela olhando a documentação do Censo) e executar um [join](https://www.qgistutorials.com/pt_BR/docs/performing_table_joins.html).

Contudo, observe que os mapas da apresentação utilizam distritos ou bairros como unidades espaciais e não setores censitários, enquanto os dados do Censo 2010 são apresentados por setor censitário. A escolha por trabalhar com distritos e bairros se deu pela maior facilidade de comunicação e interpretação. 
  
A seguir, encontra-se um exemplo de como obter a renda média per capita por distrito. Os dados são inicialmente apresentados por setor censitário, porém adotaremos uma média ponderada pela população para estimar a renda per capita por distrito. Para isso, abra a [API do Python no QGIS](https://www.qgistutorials.com/en/docs/getting_started_with_pyqgis.html) e utilize o código a seguir.

```python
distritos = QgsProject.instance().mapLayersByName('INSIRA O NOME DO LAYER CONTENDO OS DISTRITOS')[0]
setores = QgsProject.instance().mapLayersByName('INSIRA O NOME DO LAYER CONTENDO OS SETORES CENSITÁRIOS')[0]

dict_distritos_pop = {}
dict_distritos_ren = {}
for setor in setores.getFeatures():
    cod_distrito = setor['CD_GEOCODD']
    # CD_GEOCODD é o nome do campo que contém o código do distrito ao qual pertence o setor censitário
    if setor['V002'] != None and setor['V009'] != None:
    # V002 é o nome do campo que contém a população por setor censitário
    # V009 é o nome do campo que contém a renda per capita por setor censitário
        if cod_distrito not in dict_distritos_pop:
            dict_distritos_pop[cod_distrito] = int(setor['V002'])
            dict_distritos_ren[cod_distrito] = int(setor['V002']) * float(setor['V009'].replace(",", "."))
        else:
            dict_distritos_pop[cod_distrito] += int(setor['V002'])
            dict_distritos_ren[cod_distrito] += int(setor['V002']) * float(setor['V009'].replace(",", "."))

dict_distritos_popren = {}
for key in dict_distritos_pop:
	dict_distritos_popren[key] = dict_distritos_ren[key] / dict_distritos_pop[key]

distritos.startEditing()
for distrito in distritos.getFeatures():
	cod_distrito = distrito['CD_GEOCODD']
	if cod_distrito in dict_distritos_popren:
		distrito['REN_MED'] = dict_distritos_popren[cod_distrito]
        # Você deve criar o campo REN_MED no layer de distritos
		distritos.updateFeature(distrito)


```
  
Caso seja necessário apresentar os dados de renda por bairros em vez de distritos, utilize a ferramenta [Dissolver](https://www.instrutorgis.com.br/qgis3-analises-espaciais-com-dados-abertos-dissolve/) e substitua os parâmetros do layer de distritos pelos parâmetros do layer de bairros no código acima.



Utilizando recursos do QGIS, como a [Intersecção de layers](https://nates-intro-to-qgis.readthedocs.io/en/latest/geoprocessing.html#intersect), você conseguirá combinar os dados de GD com os dados do Censo 2010 e obter a renda média dos adotantes de GD no país.

