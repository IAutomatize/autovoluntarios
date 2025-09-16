# README - AutoVoluntários

<div align="center">
  <h1>🤝 AutoVoluntários</h1>
  <p><strong>Sistema inteligente de gestão de escalas para voluntários de igrejas</strong></p>
  
  ![Status](https://img.shields.io/badge/Status-Frontend%20Completo-green)
  ![Next.js](https://img.shields.io/badge/Next.js-15.5.3-black)
  ![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue)
  ![Tailwind](https://img.shields.io/badge/TailwindCSS-3.x-blue)
</div>

---

## 📖 Sobre o Projeto

O **AutoVoluntários** é uma solução completa para gestão automatizada de escalas de voluntários em igrejas, focado em resolver os principais desafios de coordenação:

- 🤖 **Substituições automáticas** com sistema de pontuação inteligente
- 📱 **Notificações em tempo real** para todas as partes interessadas  
- 🎯 **Algoritmo de adequação** que considera disponibilidade, preferências e rotação justa
- 📊 **Dashboard completo** com métricas e relatórios detalhados
- ⚙️ **Configurações flexíveis** para adaptar às necessidades de cada igreja

### 🎯 Principais Funcionalidades

- **Gestão de Voluntários**: Perfis completos com habilidades, disponibilidade e preferências
- **Calendário de Escalas**: Visualização em grade e calendário com navegação intuitiva
- **Sistema de Substituições**: Processo automatizado com pontuação de candidatos
- **Notificações Inteligentes**: Alertas contextuais e em tempo real
- **Relatórios e Analytics**: Métricas de participação, trends e insights
- **Configurações Avançadas**: Personalização de regras e comportamentos

---

## 🚀 Status Atual

### ✅ **Frontend Completo**
- Interface totalmente funcional com dados mock
- Todas as páginas implementadas e navegáveis
- Componentes reutilizáveis e design system consistente
- Responsive design para mobile e desktop
- Pronto para integração com backend

### 🔄 **Backend em Desenvolvimento**
- Documentação técnica completa disponível
- Schema de banco de dados definido
- Arquitetura e estrutura planejadas
- APIs especificadas e priorizadas

---

## 🛠️ Tecnologias Utilizadas

### **Frontend**
- **Next.js 15.5.3** - Framework React com App Router
- **TypeScript** - Tipagem estática para maior robustez
- **Tailwind CSS** - Estilização utilitária e responsiva
- **Lucide React** - Biblioteca de ícones consistente
- **React Query** - Gerenciamento de estado e cache (preparado)
- **Headless UI** - Componentes acessíveis

### **Backend (Planejado)**
- **Node.js + Express** - Runtime e framework web
- **PostgreSQL** - Banco de dados relacional
- **Prisma ORM** - Type-safe database access
- **Redis** - Cache e filas de processamento
- **JWT** - Autenticação e autorização

---

## 📁 Estrutura do Projeto

```
autovoluntarios/
├── frontend/                    # Aplicação Next.js (✅ Completo)
│   ├── src/
│   │   ├── app/                # Páginas (App Router)
│   │   │   ├── page.tsx       # Dashboard principal
│   │   │   ├── escalas/       # Gestão de escalas
│   │   │   ├── voluntarios/   # Gestão de voluntários
│   │   │   ├── substituicoes/ # Sistema de substituições
│   │   │   ├── relatorios/    # Analytics e relatórios
│   │   │   └── configuracoes/ # Configurações do sistema
│   │   ├── components/        # Componentes reutilizáveis
│   │   ├── types/            # Definições TypeScript
│   │   ├── lib/              # Utilitários e helpers
│   │   └── constants/        # Configurações e constantes
│   └── package.json
├── backend/                     # API Node.js (🔄 Planejado)
│   └── [Estrutura em desenvolvimento]
└── Docs/                       # Documentação do projeto
    ├── planejamento/          # Documentação original
    ├── status-desenvolvimento.md # Status detalhado
    ├── backend-roadmap.md     # Guia de implementação
    └── README.md              # Este arquivo
```

---

## 🚀 Como Executar

### **Pré-requisitos**
- Node.js 18+ 
- npm ou yarn
- Git

### **Frontend (Disponível)**
```bash
# 1. Clone o repositório
git clone [url-do-repositorio]
cd autovoluntarios

# 2. Instale as dependências do frontend
cd frontend
npm install

# 3. Execute o servidor de desenvolvimento
npm run dev

# 4. Acesse no navegador
# http://localhost:3000 (ou http://localhost:3001 se 3000 estiver ocupado)
```

### **Backend (Em desenvolvimento)**
```bash
# Instruções completas disponíveis em:
# Docs/backend-roadmap.md
```

---

## 📸 Screenshots

### Dashboard Principal
![Dashboard](./frontend/public/screenshot-dashboard.png)
*Visão geral com KPIs, turnos críticos e feed de atividades*

### Gestão de Escalas  
![Escalas](./frontend/public/screenshot-escalas.png)
*Calendário interativo com visualização em grade e mensal*

### Sistema de Substituições
![Substituições](./frontend/public/screenshot-substituicoes.png)
*Processo automatizado com pontuação de candidatos*

---

## 🎯 Roadmap

### **Concluído ✅**
- [x] Interface completa do frontend
- [x] Sistema de tipos TypeScript
- [x] Componentes reutilizáveis
- [x] Layout responsivo
- [x] Navegação e roteamento
- [x] Dados mock para demonstração

### **Em Desenvolvimento 🔄**
- [ ] Backend API com Node.js
- [ ] Banco de dados PostgreSQL
- [ ] Sistema de autenticação
- [ ] Algoritmo de pontuação em produção
- [ ] Notificações em tempo real

### **Próximas Funcionalidades 🔮**
- [ ] PWA (Progressive Web App)
- [ ] Notificações push nativas
- [ ] Integração com calendários externos
- [ ] App mobile (React Native)
- [ ] Sistema de check-in/check-out
- [ ] Relatórios avançados com BI

---

## 🤝 Como Contribuir

1. **Fork** o projeto
2. **Clone** seu fork localmente
3. **Crie uma branch** para sua feature (`git checkout -b feature/nova-funcionalidade`)
4. **Commit** suas mudanças (`git commit -m 'Add: nova funcionalidade'`)
5. **Push** para a branch (`git push origin feature/nova-funcionalidade`)
6. **Abra um Pull Request**

### **Convenções de Commit**
- `feat:` Nova funcionalidade
- `fix:` Correção de bug
- `docs:` Atualização de documentação
- `style:` Mudanças de formatação
- `refactor:` Refatoração de código
- `test:` Adição ou correção de testes

---

## 📋 Licença

Este projeto está sob a licença MIT. Veja o arquivo `LICENSE` para mais detalhes.

---

## 📞 Suporte

Para dúvidas, sugestões ou suporte:

- 📧 **Email**: [seu-email@exemplo.com]
- 💬 **Issues**: Use a aba Issues do GitHub
- 📖 **Documentação**: Veja a pasta `Docs/` para guias detalhados

---

## 🙏 Agradecimentos

Este projeto foi desenvolvido pensando nas necessidades reais de coordenadores de voluntários em igrejas, com foco em automatizar processos repetitivos e melhorar a experiência de todos os envolvidos.

**Funcionalidades inspiradas por:**
- Feedback de líderes de ministérios
- Desafios reais de coordenação
- Best practices de gestão de equipes
- Experiência de usuário em aplicativos modernos

---

<div align="center">
  <p>Feito com ❤️ para facilitar o trabalho voluntário nas igrejas</p>
  
  [![Frontend Demo](https://img.shields.io/badge/🌐_Frontend_Demo-Disponível-green)](http://localhost:3000)
  [![Documentação](https://img.shields.io/badge/📖_Docs-Completa-blue)](./Docs/)
  [![Status](https://img.shields.io/badge/📊_Status-Em_Desenvolvimento-yellow)](./Docs/status-desenvolvimento.md)
</div>