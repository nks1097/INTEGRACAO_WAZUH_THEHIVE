# Integração Wazuh + TheHive

Link de referência do Wazuh para integração: [Using Wazuh and TheHive for threat protection and incident response](https://wazuh.com/blog/using-wazuh-and-thehive-for-threat-protection-and-incident-response/)
video de referencia: https://www.youtube.com/watch?v=03Q51z6tKHs , https://www.youtube.com/watch?v=EbMBtkV_R_w&list=PLHjuPxrwcdsZub-nvo_yKgCu-KwSn2qTn&index=17
---

## Pré-requisitos
* Acesso administrador (root/sudo) ao servidor via SSH onde o Wazuh Manager está instalado.
* Acesso administrador à interface web do TheHive.

---

## Passo 1: Gerar a API Key no TheHive

Para que o Wazuh consiga enviar dados para o TheHive, ele precisa de uma chave de autenticação (API Key).

1. Acesse a interface web do TheHive (ex: `http://192.168.0.107:9000`).
2. Acesse a aba de **Usuários** (conforme mostrado nas imagens de apoio).

   ![TheHive Lista de Usuários](imagens/thehive-lista-usuarios.png)

3. Selecione o usuário que será responsável pela integração (ex: `natanael`, que possui o perfil `org-admin` ou `analyst`).

   ![TheHive Previsualizacao Usuário](imagens/thehive-previsualizacao-usuario.png)

4. Na página de detalhes do usuário, localize a seção **Chave API** (API Key) e clique em **Criar Chave API** (Create API Key).
5. Clique em **Revelar** (Reveal) e copie a chave gerada (ex: `sY59XDceTK6...`). Salve-a temporariamente em um bloco de notas.

   ![TheHive API Key Reveal](imagens/thehive-api-key-reveal.png)

---

## Passo 2: Instalar Dependências no Servidor Wazuh

Acesse o terminal do seu servidor Wazuh (via SSH) e execute os comandos abaixo para garantir que o Python e a biblioteca de comunicação com o TheHive estejam instalados corretamente.

1. Atualize os pacotes do sistema:
   ```bash
   sudo yum update -y
   ```
2. Instale o Python 3 (caso não esteja instalado):
   ```bash
   sudo yum install python3 -y
   ```
3. Instale o módulo `thehive4py` utilizando o PIP interno do próprio framework do Wazuh (isso evita erros de compatibilidade, conforme explicado no vídeo):
   ```bash
   sudo /var/ossec/framework/python/bin/pip3 install thehive4py==1.8.1
   ```

---

## Passo 3: Criar os Scripts de Integração

O Wazuh utiliza scripts localizados na pasta `integrations` para encaminhar os alertas.

1. Navegue até o diretório de integrações do Wazuh:
   ```bash
   cd /var/ossec/integrations/
   ```
2. Crie o arquivo Python que fará a integração:
   ```bash
   sudo nano custom-w2thive.py
   ```
3. Cole o código Python de integração fornecido pelo repositório base (visto no vídeo).
4. **Atenção à configuração de Threshold (Imagem 8):** Dentro do código Python, localize a seção `# Global vars`. A variável `lvl_threshold=8` indica que apenas alertas do Wazuh de nível 8 ou superior serão enviados ao TheHive. Você pode alterar esse número conforme a necessidade do seu laboratório.
5. Salve e saia do arquivo (Ctrl+O, Enter, Ctrl+X).
6. Crie o script bash de execução (wrapper):
   ```bash
   sudo nano custom-w2thive
   ```
7. Cole o código bash correspondente à integração e salve o arquivo. *(Nota: O conteúdo deste bash serve apenas para chamar o script `.py` criado acima).*

---

## Passo 4: Configurar Permissões (Extremamente Importante)

Conforme destacado no vídeo, se as permissões e o grupo dono dos arquivos estiverem errados, a integração falhará silenciosamente.

1. Dê permissão de execução (755) para ambos os arquivos criados:
   ```bash
   sudo chmod 755 /var/ossec/integrations/custom-w2thive*
   ```
2. Altere o dono e o grupo dos arquivos para `root` e `wazuh` (versões mais recentes do Wazuh usam o grupo `wazuh`, e não `ossec`):
   ```bash
   sudo chown root:wazuh /var/ossec/integrations/custom-w2thive*
   ```

---

## Passo 5: Configurar o ossec.conf no Wazuh

Agora precisamos avisar ao Wazuh Manager que a integração existe e para onde ela deve enviar os alertas.

1. Abra o arquivo de configuração principal do Wazuh:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```
2. Role o arquivo até encontrar o final do bloco `<ossec_config>` (ou procure por outras tags `<integration>`).
3. Adicione o seguinte bloco de código (baseado na Imagem 9 fornecida):
   ```xml
   <integration>
       <name>custom-w2thive</name>
       <hook_url>http://192.168.0.107:9000</hook_url>
       <api_key>COLE_AQUI_A_SUA_API_KEY_DO_THEHIVE</api_key>
       <alert_format>json</alert_format>
   </integration>
   ```
   
   ![Wazuh ossec.conf Integration](imagens/wazuh-ossec-conf-integration.png)

4. Substitua o IP `192.168.0.107` pelo IP real do seu servidor TheHive.
5. Substitua o conteúdo de `<api_key>` pela chave que você copiou no Passo 1.
6. Salve e saia do arquivo.

---

## Passo 6: Reiniciar o Wazuh e Validar

Para que o Wazuh leia a nova configuração no `ossec.conf` e carregue o script de integração, é obrigatório reiniciar o serviço.

1. Reinicie o Wazuh Manager:
   ```bash
   sudo systemctl restart wazuh-manager
   ```
2. Verifique se o serviço subiu sem erros:
   ```bash
   sudo systemctl status wazuh-manager
   ```

### Validação:
* Vá para a interface web do TheHive, clique no menu **Alerts** (Alertas) no painel esquerdo.
* Aguarde alguns minutos ou force um erro no seu ambiente monitorado pelo Wazuh (como falhas sucessivas de login via SSH). Assim que o Wazuh gerar um alerta de nível igual ou maior que 8 (configurado no Threshold), ele aparecerá automaticamente na fila de alertas do TheHive, pronto para ser transformado em um Caso de Investigação.
