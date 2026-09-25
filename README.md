# AvaliaGestão

Sistema interno para ciclos de avaliação de gerentes, com respostas da equipe, autoavaliação, relatórios agregados e controle de acesso por perfil.

## Como funciona a autoavaliação

1. Uma pessoa de RH abre **Autoavaliações**, escolhe um gerente ativo, um período aberto e a validade do link.
2. O sistema cria um link aleatório que pode ser usado uma vez. Copie o link mostrado na tela e envie ao gerente por um canal privado.
3. O gerente abre o link, responde aos critérios de gestão e envia a autoavaliação. Comentários são opcionais.
4. A resposta fica associada ao gerente e ao período, numa coleção separada das avaliações da equipe. Isso é necessário para comparar a percepção do gerente com a equipe; a autoavaliação não é anônima em relação ao gerente.
5. O relatório protege os números da equipe até atingir o limite mínimo de respostas configurado, cinco por padrão. Links pendentes podem ser cancelados por RH.

O código secreto do link é guardado como hash. Ele só é exibido uma vez, logo após a geração. Se perder o link, cancele-o e gere outro.

## Perfis e senhas

- **RH/DP:** gerencia os ciclos, gerentes, setores e critérios; gera links individuais; consulta relatórios agregados.
- **Administrador:** inclui RH e outros administradores em **Usuários**, acompanha o último acesso, ativa ou desativa contas, redefine senhas e exclui contas definitivamente.
- Contas criadas ou redefinidas pela área de usuários recebem uma senha temporária aleatória de 12 caracteres. No primeiro acesso, é obrigatório criar uma senha pessoal com pelo menos 8 caracteres, incluindo maiúscula, minúscula, número e símbolo.
- Qualquer pessoa autenticada pode trocar a própria senha pelo menu do perfil, confirmando primeiro a senha atual.
- A exclusão de uma conta remove seu login, encerra as sessões e mantém o registro de auditoria. A própria conta e o último administrador ativo não podem ser excluídos ou desativados.
- Gerentes sem avaliações e sem links podem ser excluídos. Se houver histórico, desative o gerente: ele sai da lista de avaliação e os relatórios e comentários anteriores ficam preservados.

## Avaliação anônima da equipe

- Durante um período aberto, funcionários acessam o formulário público em `/avaliar`, sem criar conta. O RH pode copiar esse link no dashboard e compartilhá-lo no mural, em um QR code ou no canal interno.
- O RH/DP vê os comentários sem nome ou matrícula em **Comentários**, filtrando por gerente e período. O formulário alerta que não se deve escrever dados que identifiquem alguém.
- A média da equipe e a distribuição das notas aparecem depois do limite mínimo configurado, cinco respostas por gerente por padrão. Reduzir esse limite para testar também reduz a proteção contra identificar respostas em equipes pequenas.
- Como o formulário é aberto e não pede login ou código individual, ele não consegue comprovar que cada funcionário respondeu exatamente uma vez. O sistema evita duplicar o reenvio do mesmo formulário, mas um link recarregado pode gerar uma nova resposta. Para controlar uma resposta por pessoa sem gravar nomes, seria necessário distribuir códigos anônimos individuais.

Para criar o primeiro ADMIN do Firebase, o script de bootstrap usa as credenciais do Firebase Admin e exige `BOOTSTRAP_ADMIN_ALLOW=true`, `BOOTSTRAP_ADMIN_EMAIL` e `BOOTSTRAP_ADMIN_NAME`. Se `BOOTSTRAP_ADMIN_PASSWORD` não for informada, ele gera uma senha forte temporária, mostra-a uma única vez no terminal e exige que seja trocada no primeiro acesso. O script só cria a conta se ainda não houver administrador ativo.

## Desenvolvimento local

1. Instale Node.js 22 ou superior e as dependências com `npm ci`.
2. Copie `.env.example` para `.env.local` e configure o Firebase Emulator Suite local.
3. Inicie os emuladores com `npm run emulators` e a aplicação com `npm run dev`.
4. Para carregar dados fictícios, configure `RH_SEED_PASSWORD` e execute `npm run seed`. O script recusa rodar sem o emulador explicitamente habilitado.

`npm run build` gera a versão otimizada. As regras do Firestore bloqueiam o acesso direto pelo navegador; a aplicação lê e grava dados pelo servidor com Firebase Admin.

## Hospedar na rede interna com Raspberry Pi

### Arquitetura preparada

O Raspberry Pi hospeda o site Next.js em Docker na porta HTTP 8081; o GestaoVista permanece no Nginx da porta 80. Contas, autenticação e avaliações continuam no Firebase Cloud. Assim, o Pi precisa de internet para falar com o Firebase, mas os funcionários acessam o site pela rede interna. Para operar sem internet e guardar tudo no Pi, seria necessário substituir Firebase Auth/Firestore por serviços locais e migrar os dados.

O contêiner usa Node.js 24 e a saída standalone do Next.js. O Firebase Admin SDK atual requer Node.js 22 ou superior; Node.js 24 é uma linha LTS atual. O guia oficial do Next.js recomenda um proxy reverso ao hospedar a aplicação por conta própria. [Node.js releases](https://nodejs.org/en/about/previous-releases), [Firebase Admin setup](https://firebase.google.com/docs/admin/setup), [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting).

### Preparar o Pi

1. Use um Raspberry Pi com Raspberry Pi OS de 64 bits e Docker com Docker Compose instalados.
2. Reserve um IP local para o Pi no roteador, por exemplo `10.66.10.98`.
3. Configure um projeto Firebase de produção: Authentication com e-mail/senha, Firestore e as credenciais do Firebase Admin SDK. Nunca use credenciais de serviço dentro do navegador.
4. Copie `.env.pi.example` para `.env.pi` e preencha o IP reservado (`SITE_BIND_IP`), a URL HTTP, a configuração pública do Firebase e os três valores privados do Firebase Admin. Proteja esse arquivo e não o envie ao Git.
5. A partir da pasta do projeto, construa e inicie os contêineres:

   ```sh
   docker compose --env-file .env.pi up -d --build
   docker compose --env-file .env.pi ps
   ```

6. Cadastre o primeiro administrador no contêiner, usando o mesmo projeto Firebase de produção. O comando gera uma senha temporária e a mostra uma única vez no terminal; entregue-a por um canal privado:

   ```sh
   docker compose --env-file .env.pi run --rm --no-deps \
     -e BOOTSTRAP_ADMIN_ALLOW=true \
     -e BOOTSTRAP_ADMIN_EMAIL=admin@empresa.com \
     -e BOOTSTRAP_ADMIN_NAME="Administrador inicial" \
     app node scripts/bootstrap-admin.cjs
   ```
7. Autorize apenas a sub-rede da produção no firewall do Pi/rede. O AvaliaGestão publica HTTP na porta `SITE_APP_PORT` (8081 por padrão), somente no IP definido em `SITE_BIND_IP`. Não crie redirecionamento de portas do roteador para a Internet.

Esta configuração usa HTTP sem criptografia para evitar instalar uma CA interna nos computadores. Use-a apenas numa rede interna controlada: senhas, sessões e dados de avaliações podem ser observados por alguém com acesso à rede. Para proteger esses dados, a empresa deve disponibilizar HTTPS com certificado confiável.

Acesse `http://10.66.10.98:8081` (ou o IP e a porta definidos em `.env.pi`). Os serviços reiniciam automaticamente após reinicialização do Raspberry. Consulte o estado com `docker compose --env-file .env.pi logs -f app`.

### Dados e cópias de segurança

Com a configuração padrão, os dados ficam no Firebase Cloud e não no cartão SD do Raspberry. Configure as cópias de segurança e o controle de acesso no projeto Firebase. Os contêineres Docker guardam apenas a aplicação.

## Segurança já aplicada

- Senhas e sessões não são guardadas em texto claro pelo AvaliaGestão; os links de autoavaliação novos são guardados como hash.
- A troca de senha exige autenticação recente; ao trocar, as sessões antigas são revogadas.
- As regras Firestore negam leitura e gravação diretas do navegador; só o servidor com credencial Admin tem acesso.
- A sessão do servidor é HTTP-only e recebe `Secure` quando a conexão chega por HTTPS. Na configuração HTTP da rede interna, navegadores não recebem essa proteção de transporte.
- Cabeçalhos reduzem incorporação em frames, MIME sniffing e permissões de dispositivos que o sistema não usa.
- `.gitignore` e `.dockerignore` excluem configurações locais, chaves de serviço, dados do emulador e logs.
