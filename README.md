# Trabalho-Fabiana-
Entrega do Projeto Integrador – Sistema Digital de Controle de Horas Extras

/*
  PROJETO INTEGRADOR: Sistema Digital de Controle de Horas Extras
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h> 

// --- Definições Globais ---
#define MAX_REGISTROS 1000
#define MAX_USUARIOS 100
#define ARQUIVO_HORAS "horas_extras.dat" // Base local de cada usuário
#define ARQUIVO_USUARIOS "usuarios.dat" // Base local de cada usuário

// --- Estruturas de Dados ---

/* Enum para os Tipos de Hora Extra (Baseado na CLT) */
typedef enum {
    TIPO_NORMAL,           // Dia de semana (Adicional 50%)
    TIPO_FDS_FERIADO,      // Fim de Semana/Feriado (Adicional 100%)
    TIPO_NORMAL_NOTURNA,   // Dia de semana Noturna (Adicional 80% = 1.2 * 1.5)
    TIPO_FDS_NOTURNA       // FDS/Feriado Noturna (Adicional 140% = 1.2 * 2.0)
} TipoHora;

/* Estrutura para os registros de horas (MODIFICADA) */
typedef struct {
    int idRegistro;            // ID único para cada lançamento
    char idFuncionario[50];    // Login do funcionário que registrou
    time_t timestamp;          // Guarda a data exata do registro (via time.h)
    float horas;               // Quantidade de horas (ex: 2.5)
    char status[20];           // "Pendente", "Aprovado", "Rejeitado", "Exportado"
    TipoHora tipo;             // Guarda o tipo da hora extra (CLT)
    char justificativa[150];   // <-- NOVO: Motivo da rejeição pelo gestor
} RegistroHoraExtra;

/* Estrutura para os Usuários do sistema (Sem mudança) */
typedef enum { FUNCIONARIO, GESTOR, ADMIN } TipoUsuario;

typedef struct {
    char login[50];
    char senha[50];
    TipoUsuario tipo;
} Usuario;


// --- Variáveis Globais (Base de Dados em Memória) ---
RegistroHoraExtra registros[MAX_REGISTROS];
int numRegistros = 0;
int proximoIdRegistro = 1;

Usuario usuarios[MAX_USUARIOS];
int numUsuarios = 0;

Usuario usuarioLogado; // Guarda quem está a usar o sistema
int logado = 0;        // Flag 0 = Ninguém logado, 1 = Logado

// --- Protótipos das Funções ---

// Funções de Utilidade
void limparBufferEntrada();
void lerString(char *destino, int tamanho);
void pausar();
const char* getTipoUsuarioString(TipoUsuario tipo);
const char* getTipoHoraString(TipoHora tipo); 
void formatarTimestamp(time_t ts, char* buffer, int tam); 
void limparTela();

// Funções de Persistência (Salvar/Carregar)
void salvarHoras();
void carregarHoras();
void salvarUsuarios();
void carregarUsuarios();

// Funções de Lógica Principal
void inicializarAdmin(); 
int fazerLogin();
void fazerLogout();

// Menus (Portais)
void menuAdmin();
void menuGestor();
void menuFuncionario();

// Funções de Ação (Separadas por Função)
void criarUsuario();
void listarUsuarios();
void removerUsuario(); 
void registrarMinhaHora(); 
void listarMinhasHoras(); 
void removerMinhaHora(); 
void exportarHorasPendentes(); 
void listarRegistros(const char* filtroStatus); 
void aprovarReprovarHora(); 
void calcularTotalHorasFuncionario(); 
void importarHorasCSV(); 


// --- Função Principal (main) ---
int main() {
    // 1. Carrega os dois bancos de dados (LOCAIS)
    carregarUsuarios();
    carregarHoras(); // Se o .dat for antigo, pode dar erro aqui.

    // 2. Garante que o Admin Mestre existe
    inicializarAdmin();

    printf("===========================================\n");
    printf("  Sistema de Controle de Horas Extras (C)\n");
    printf("  (Protótipo Lógico - v5.2)\n"); // Versão atualizada
    printf("===========================================\n");
    printf("Bem-vindo. Por favor, faça o login.\n");

    // 3. Tenta fazer o login
    if (fazerLogin()) {

        // 4. Se o login for bem-sucedido, direciona para o portal correto
        switch (usuarioLogado.tipo) {
            case FUNCIONARIO:
                menuFuncionario();
                break;
            case GESTOR:
                menuGestor();
                break;
            case ADMIN:
                menuAdmin();
                break;
        }
    } else {
        printf("\nLogin ou senha incorretos. O programa será encerrado.\n");
    }

    printf("\nPrograma encerrado.\n");
    return 0;
}


// --- Implementação das Funções de Utilidade ---

void limparTela() {
    system("clear || cls");
}

void limparBufferEntrada() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF);
}

void lerString(char *destino, int tamanho) {
    fgets(destino, tamanho, stdin);
    destino[strcspn(destino, "\n")] = 0;
}

void pausar() {
    printf("\nPressione ENTER para continuar...");
    getchar();
}

const char* getTipoUsuarioString(TipoUsuario tipo) {
    switch (tipo) {
        case FUNCIONARIO: return "Funcionario";
        case GESTOR: return "Gestor";
        case ADMIN: return "Admin";
        default: return "Desconhecido";
    }
}

const char* getTipoHoraString(TipoHora tipo) {
    switch (tipo) {
        case TIPO_NORMAL: return "Normal (50%)";
        case TIPO_FDS_FERIADO: return "FDS/Feriado (100%)";
        case TIPO_NORMAL_NOTURNA: return "Normal Noturna (80%)";
        case TIPO_FDS_NOTURNA: return "FDS/Feriado Noturna (140%)";
        default: return "N/A";
    }
}

void formatarTimestamp(time_t ts, char* buffer, int tam) {
    struct tm *infoTempo;
    infoTempo = localtime(&ts); 
    strftime(buffer, tam, "%Y-%m-%d", infoTempo); 
}


// --- Implementação das Funções de Persistência ---
// (AVISO: A ESTRUTURA MUDOU, DELETE O 'horas_extras.dat' ANTIGO)

void salvarHoras() {
    FILE *f = fopen(ARQUIVO_HORAS, "wb");
    if (f == NULL) {
        printf("Erro fatal ao salvar horas locais.\n"); return;
    }
    fwrite(&numRegistros, sizeof(int), 1, f);
    fwrite(&proximoIdRegistro, sizeof(int), 1, f);
    fwrite(registros, sizeof(RegistroHoraExtra), numRegistros, f);
    fclose(f);
}

void carregarHoras() {
    FILE *f = fopen(ARQUIVO_HORAS, "rb");
    if (f == NULL) {
        printf("Info: Arquivo %s não encontrado. (Horas locais)\n", ARQUIVO_HORAS);
        return;
    }

    // Leitura dos contadores
    if (fread(&numRegistros, sizeof(int), 1, f) != 1) {
        printf("Info: Arquivo %s vazio ou corrompido.\n", ARQUIVO_HORAS);
        numRegistros = 0;
        fclose(f);
        return;
    }
    if (fread(&proximoIdRegistro, sizeof(int), 1, f) != 1) {
        printf("Info: Arquivo %s corrompido (ID).\n", ARQUIVO_HORAS);
        proximoIdRegistro = 1;
        fclose(f);
        return;
    }

    // Leitura dos registros
    if (numRegistros > 0) {
        int lidos = fread(registros, sizeof(RegistroHoraExtra), numRegistros, f);
        if (lidos != numRegistros) {
            printf("AVISO: Arquivo %s pode estar corrompido ou é de uma versão antiga.\n", ARQUIVO_HORAS);
            printf("Esperado ler %d registros, mas leu %d.\n", numRegistros, lidos);
            // Tenta continuar mesmo assim, mas pode dar problema.
            numRegistros = lidos; // Ajusta para o que foi lido
        }
    }

    fclose(f);
}

void salvarUsuarios() {
    FILE *f = fopen(ARQUIVO_USUARIOS, "wb");
    if (f == NULL) {
        printf("Erro fatal ao salvar usuários locais.\n"); return;
    }
    fwrite(&numUsuarios, sizeof(int), 1, f);
    fwrite(usuarios, sizeof(Usuario), numUsuarios, f);
    fclose(f);
}

void carregarUsuarios() {
    FILE *f = fopen(ARQUIVO_USUARIOS, "rb");
    if (f == NULL) {
        printf("Info: Arquivo %s não encontrado. (Usuários locais)\n", ARQUIVO_USUARIOS);
        return;
    }
    fread(&numUsuarios, sizeof(int), 1, f);
    fread(usuarios, sizeof(Usuario), numUsuarios, f);
    fclose(f);
}

// --- Implementação da Lógica Principal ---

void inicializarAdmin() {
    if (numUsuarios == 0) {
        printf("Sistema não inicializado. Criando usuário Admin Mestre...\n");
        strcpy(usuarios[0].login, "admin");
        strcpy(usuarios[0].senha, "admin");
        usuarios[0].tipo = ADMIN;
        numUsuarios = 1;
        salvarUsuarios();
        printf("Usuário 'admin' (senha 'admin') criado com sucesso.\n");
    }
}

int fazerLogin() {
    char login[50];
    char senha[50];

    printf("Login: ");
    lerString(login, 50);
    printf("Senha: ");
    lerString(senha, 50);

    for (int i = 0; i < numUsuarios; i++) {
        if (strcmp(usuarios[i].login, login) == 0 && strcmp(usuarios[i].senha, senha) == 0) {
            usuarioLogado = usuarios[i]; 
            logado = 1;
            return 1; // Sucesso
        }
    }
    return 0; // Falha
}

void fazerLogout() {
    logado = 0;
    memset(&usuarioLogado, 0, sizeof(Usuario)); 
    printf("Logout efetuado com sucesso.\n");
}


// --- Implementação dos MENUS (Portais) ---

void menuAdmin() {
    int opcao;
    do {
        limparTela();
        printf("===========================================\n");
        printf("  Portal do Administrador\n");
        printf("  Usuário: %s\n", usuarioLogado.login);
        printf("===========================================\n");
        printf("1. Criar Novo Usuário (Gestor ou Funcionário)\n");
        printf("2. Listar Todos os Usuários\n");
        printf("3. Remover Usuário\n"); 
        printf("0. Sair (Logout)\n");
        printf("\nEscolha uma opção: ");

        if (scanf("%d", &opcao) != 1) {
            opcao = -1;
        }
        limparBufferEntrada();

        switch (opcao) {
            case 1: criarUsuario(); break;
            case 2: listarUsuarios(); break;
            case 3: removerUsuario(); break; 
            case 0: fazerLogout(); break;
            default: printf("Opção inválida.\n");
        }
        if(opcao != 0) pausar();
    } while (opcao != 0);
}

void menuGestor() {
    int opcao;
    do {
        limparTela();
        printf("===========================================\n");
        printf("  Portal do Gestor\n");
        printf("  Usuário: %s\n", usuarioLogado.login);
        printf("===========================================\n");
        printf("1. Listar Horas Pendentes de Aprovação\n");
        printf("2. Aprovar/Rejeitar Hora (Com Justificativa)\n"); // MODIFICADO
        printf("3. Gerar Relatório Financeiro (Por Funcionário)\n");
        printf("4. Listar Todos os Registros (Histórico Geral)\n");
        printf("5. Importar Arquivo CSV de Horas\n");
        printf("0. Sair (Logout)\n");
        printf("\nEscolha uma opção: ");

        if (scanf("%d", &opcao) != 1) {
            opcao = -1;
        }
        limparBufferEntrada();

        switch (opcao) {
            case 1: 
                limparTela(); 
                printf("--- Horas Pendentes de Aprovação (Base Local) ---\n");
                listarRegistros("Pendente");
                break;
            case 2: 
                limparTela();
                aprovarReprovarHora(); 
                break;
            case 3: 
                limparTela();
                calcularTotalHorasFuncionario(); 
                break;
            case 4: 
                limparTela();
                printf("--- Todos os Registros (Base Local) ---\n");
                listarRegistros(NULL); // NULL = sem filtro
                break;
            case 5: 
                limparTela();
                importarHorasCSV();
                break;
            case 0: fazerLogout(); break;
            default: printf("Opção inválida.\n");
        }
        if(opcao != 0) pausar();
    } while (opcao != 0);
}

void menuFuncionario() {
    int opcao;
    do {
        limparTela();
        printf("===========================================\n");
        printf("  Portal do Funcionário\n");
        printf("  Usuário: %s\n", usuarioLogado.login);
        printf("===========================================\n");
        printf("1. Registrar Minhas Horas Extras\n");
        printf("2. Listar Minhas Horas (Ver Justificativas)\n"); // MODIFICADO
        printf("3. Remover um Registro Pendente\n"); 
        printf("4. Exportar Horas Pendentes para CSV\n"); 
        printf("0. Sair (Logout)\n");
        printf("\nEscolha uma opção: ");

        if (scanf("%d", &opcao) != 1) {
            opcao = -1;
        }
        limparBufferEntrada();

        switch (opcao) {
            case 1: 
                limparTela();
                registrarMinhaHora(); 
                break;
            case 2: 
                limparTela();
                listarMinhasHoras(); 
                break;
            case 3: 
                limparTela();
                removerMinhaHora();
                break;
            case 4: 
                limparTela();
                exportarHorasPendentes();
                break;
            case 0: fazerLogout(); break;
            default: printf("Opção inválida.\n");
        }
        if(opcao != 0) pausar();
    } while (opcao != 0);
}

// --- Implementação das Funções de Ação ---

void criarUsuario() {
    if (numUsuarios >= MAX_USUARIOS) {
        printf("Erro: Limite de usuários atingido.\n"); return;
    }
    int i = numUsuarios;
    int tipoInt;

    printf("\n--- Criar Novo Usuário ---\n");
    printf("Digite o login (ex: jose.silva): ");
    lerString(usuarios[i].login, 50);
    printf("Digite a senha: ");
    lerString(usuarios[i].senha, 50);

    printf("Digite o tipo (1 = Funcionário, 2 = Gestor): ");
    while(scanf("%d", &tipoInt) != 1 || (tipoInt != 1 && tipoInt != 2)) {
        printf("Entrada inválida. Digite 1 ou 2: ");
        limparBufferEntrada();
    }

    usuarios[i].tipo = (tipoInt == 1) ? FUNCIONARIO : GESTOR;

    numUsuarios++;
    salvarUsuarios(); 

    printf("\nUsuário '%s' (%s) criado com sucesso.\n", 
           usuarios[i].login, getTipoUsuarioString(usuarios[i].tipo));
}

void listarUsuarios() {
    limparTela();
    printf("--- Lista de Usuários do Sistema ---\n");
    printf("Total: %d\n", numUsuarios);
    printf("--------------------------------------------------\n");
    printf("%-20s | %-15s\n", "Login", "Função (Role)");
    printf("--------------------------------------------------\n");
    for (int i = 0; i < numUsuarios; i++) {
        printf("%-20s | %-15s\n", 
               usuarios[i].login, 
               getTipoUsuarioString(usuarios[i].tipo));
    }
    printf("--------------------------------------------------\n");
}

void removerUsuario() {
    char loginRemover[50];
    int indiceEncontrado = -1;

    limparTela();
    printf("\n--- Remover Usuário ---\n");

    listarUsuarios();
    printf("--------------------------------------------------\n");
    printf("AVISO: Não é possível remover o usuário 'admin'.\n");

    printf("\nDigite o login do usuário que deseja remover: ");
    lerString(loginRemover, 50);

    if (strcmp(loginRemover, "admin") == 0) {
        printf("Erro: Não é permitido remover o usuário 'admin' mestre.\n");
        return;
    }

    for (int i = 0; i < numUsuarios; i++) {
        if (strcmp(usuarios[i].login, loginRemover) == 0) {
            indiceEncontrado = i;
            break;
        }
    }

    if (indiceEncontrado == -1) {
        printf("Erro: Usuário com login '%s' não encontrado.\n", loginRemover);
        return;
    }

    char escolha;
    printf("Tem certeza que deseja remover '%s' (%s)? (s/n): ", 
           usuarios[indiceEncontrado].login,
           getTipoUsuarioString(usuarios[indiceEncontrado].tipo));
    scanf(" %c", &escolha);
    limparBufferEntrada();

    if (escolha != 's' && escolha != 'S') {
        printf("Remoção cancelada.\n");
        return;
    }

    for (int i = indiceEncontrado; i < numUsuarios - 1; i++) {
        usuarios[i] = usuarios[i + 1]; 
    }

    numUsuarios--;
    salvarUsuarios();

    printf("Usuário '%s' removido com sucesso.\n", loginRemover);
}


/* (Ação do FUNCIONÁRIO) - MODIFICADO */
void registrarMinhaHora() {
    if (numRegistros >= MAX_REGISTROS) {
        printf("Erro: Limite de registros atingido.\n"); return;
    }
    int i = numRegistros;
    int tipoInt;

    printf("\n--- Registrar Minha Hora ---\n");

    registros[i].idRegistro = proximoIdRegistro;
    strcpy(registros[i].status, "Pendente"); 
    strcpy(registros[i].idFuncionario, usuarioLogado.login); 
    registros[i].timestamp = time(NULL); 
    strcpy(registros[i].justificativa, ""); // <-- NOVO: Inicializa a justificativa

    printf("Funcionário: %s (Automático)\n", registros[i].idFuncionario);

    printf("Tipo de Hora Extra (Base CLT):\n");
    printf("  1. Normal (Dia de semana, 50%%)\n");
    printf("  2. Fim de Semana ou Feriado (100%%)\n");
    printf("  3. Normal Noturna (Dia de semana, 22h-05h, 80%%)\n");
    printf("  4. Fim de Semana/Feriado Noturna (22h-05h, 140%%)\n");
    printf("Escolha o tipo: ");

    while(scanf("%d", &tipoInt) != 1 || (tipoInt < 1 || tipoInt > 4)) {
        printf("Entrada inválida. Digite 1, 2, 3 ou 4: ");
        limparBufferEntrada();
    }
    limparBufferEntrada();

    registros[i].tipo = (TipoHora)(tipoInt - 1); 

    printf("Horas (ex: 2.5): ");
    while (scanf("%f", &registros[i].horas) != 1) {
        printf("Entrada inválida. Digite um número (ex: 2.5): ");
        limparBufferEntrada();
    }
    limparBufferEntrada();

    numRegistros++;
    proximoIdRegistro++;
    salvarHoras(); 

    printf("\nRegistro #%d (Tipo: %s) salvo como 'Pendente'.\n", 
           registros[i].idRegistro, getTipoHoraString(registros[i].tipo));
}

/* (Ação do FUNCIONÁRIO) - MODIFICADO */
void listarMinhasHoras() {
    printf("\n--- Meu Histórico de Horas Extras ---\n");
    int encontrados = 0;

    printf("---------------------------------------------------------------------------------------------------\n");
    printf("%-5s | %-20s | %-12s | %-30s | %-6s | %-15s\n", "ID", "Funcionário", "Data", "Tipo", "Horas", "Status");
    printf("---------------------------------------------------------------------------------------------------\n");

    for (int i = 0; i < numRegistros; i++) {
        if (strcmp(registros[i].idFuncionario, usuarioLogado.login) == 0) {

            char dataFormatada[12];
            formatarTimestamp(registros[i].timestamp, dataFormatada, 12);

            printf("%-5d | %-20s | %-12s | %-30s | %-6.1f | %-15s\n",
                   registros[i].idRegistro,
                   registros[i].idFuncionario,
                   dataFormatada, 
                   getTipoHoraString(registros[i].tipo), 
                   registros[i].horas,
                   registros[i].status);

            // <-- NOVO: Imprime a justificativa se houver
            if (strcmp(registros[i].status, "Rejeitado") == 0 && strlen(registros[i].justificativa) > 0) {
                printf("      | Motivo (Gestor): %s\n", registros[i].justificativa);
            }

            encontrados++;
        }
    }

    if (encontrados == 0) {
         printf("Você ainda não possui registros.\n");
    }
    printf("---------------------------------------------------------------------------------------------------\n");
}


/* (Ação do FUNCIONÁRIO) */
void removerMinhaHora() {
    int idBusca;
    int indiceEncontrado = -1;
    int encontrados = 0;

    limparTela();
    printf("\n--- Remover Meu Registro de Hora Pendente ---\n");
    printf("Apenas registros com status 'Pendente' podem ser removidos.\n\n");

    printf("--- Seus Registros Pendentes ---\n");
    printf("---------------------------------------------------------------------------------------------------\n");
    printf("%-5s | %-12s | %-30s | %-6s | %-15s\n", "ID", "Data", "Tipo", "Horas", "Status");
    printf("---------------------------------------------------------------------------------------------------\n");

    for (int i = 0; i < numRegistros; i++) {
        if (strcmp(registros[i].idFuncionario, usuarioLogado.login) == 0 &&
            strcmp(registros[i].status, "Pendente") == 0) {

            char dataFormatada[12];
            formatarTimestamp(registros[i].timestamp, dataFormatada, 12);

            printf("%-5d | %-12s | %-30s | %-6.1f | %-15s\n",
                   registros[i].idRegistro,
                   dataFormatada,
                   getTipoHoraString(registros[i].tipo),
                   registros[i].horas,
                   registros[i].status);
            encontrados++;
        }
    }
    printf("---------------------------------------------------------------------------------------------------\n");

    if (encontrados == 0) {
        printf("Você não possui registros 'Pendentes' para remover.\n");
        return; 
    }

    printf("\nDigite o ID do registro que deseja remover: ");
    if (scanf("%d", &idBusca) != 1) {
        printf("ID inválido.\n");
        limparBufferEntrada();
        return;
    }
    limparBufferEntrada();

    for (int i = 0; i < numRegistros; i++) {
        if (registros[i].idRegistro == idBusca) {
            if (strcmp(registros[i].idFuncionario, usuarioLogado.login) != 0) {
                printf("Erro: Você não tem permissão para remover o registro #%d.\n", idBusca);
                return;
            }
            if (strcmp(registros[i].status, "Pendente") != 0) {
                printf("Erro: O registro #%d não está 'Pendente' (Status: %s).\n", idBusca, registros[i].status);
                return;
            }
            indiceEncontrado = i;
            break;
        }
    }

    if (indiceEncontrado == -1) {
        printf("Erro: ID de registro %d não encontrado na sua lista de pendentes.\n", idBusca);
        return;
    }

    char escolha;
    char dataFormatada[12]; 
    formatarTimestamp(registros[indiceEncontrado].timestamp, dataFormatada, 12); 

    printf("Registro encontrado: [Data: %s, Horas: %.1f, Tipo: %s]\n",
           dataFormatada, 
           registros[indiceEncontrado].horas,
           getTipoHoraString(registros[indiceEncontrado].tipo));

    printf("Tem certeza que deseja remover este registro? (s/n): ");
    scanf(" %c", &escolha);
    limparBufferEntrada();

    if (escolha != 's' && escolha != 'S') {
        printf("Remoção cancelada.\n");
        return;
    }

    for (int i = indiceEncontrado; i < numRegistros - 1; i++) {
        registros[i] = registros[i + 1]; 
    }

    numRegistros--;
    salvarHoras();

    printf("Registro #%d removido com sucesso.\n", idBusca);
}


/* (Ação do FUNCIONÁRIO) */
void exportarHorasPendentes() {
    char nomeArquivo[100];
    int exportados = 0;

    printf("\n--- Exportar Minhas Horas Pendentes para CSV ---\n");
    printf("Digite um nome para o arquivo de exportação (ex: %s_export.csv): ", usuarioLogado.login);
    lerString(nomeArquivo, 100);

    FILE *f = fopen(nomeArquivo, "w"); 
    if (f == NULL) {
        printf("Erro: Não foi possível criar o arquivo '%s'.\n", nomeArquivo);
        return;
    }

    // O CSV não precisa da justificativa, pois só exporta "Pendente"
    fprintf(f, "ID,Funcionario,Timestamp,Horas,Tipo,Status\n");

    for (int i = 0; i < numRegistros; i++) {
        if (strcmp(registros[i].idFuncionario, usuarioLogado.login) == 0 &&
            strcmp(registros[i].status, "Pendente") == 0) {

            fprintf(f, "%d,%s,%ld,%.2f,%d,%s\n",
                    registros[i].idRegistro,
                    registros[i].idFuncionario,
                    (long)registros[i].timestamp,
                    registros[i].horas,
                    (int)registros[i].tipo,
                    registros[i].status
            );

            strcpy(registros[i].status, "Exportado");
            exportados++;
        }
    }

    fclose(f);

    if (exportados > 0) {
        printf("\n%d registros pendentes foram exportados com sucesso para '%s'.\n", exportados, nomeArquivo);
        printf("O status local deles foi alterado para 'Exportado'.\n");
        salvarHoras(); 
    } else {
        printf("\nNenhum registro 'Pendente' encontrado para exportar.\n");
        remove(nomeArquivo); 
    }
}


/* (Ação do GESTOR) - MODIFICADO */
void listarRegistros(const char* filtroStatus) {
    int encontrados = 0;
    if (numRegistros == 0) {
        printf("Nenhum registro encontrado na base local.\n");
        return;
    }

    printf("---------------------------------------------------------------------------------------------------\n");
    printf("%-5s | %-20s | %-12s | %-30s | %-6s | %-15s\n", "ID", "Funcionário", "Data", "Tipo", "Horas", "Status");
    printf("---------------------------------------------------------------------------------------------------\n");

    for (int i = 0; i < numRegistros; i++) {
        if (strcmp(registros[i].status, "Exportado") == 0) {
            continue;
        }

        if (filtroStatus == NULL || strcmp(registros[i].status, filtroStatus) == 0) {

            char dataFormatada[12];
            formatarTimestamp(registros[i].timestamp, dataFormatada, 12);

            printf("%-5d | %-20s | %-12s | %-30s | %-6.1f | %-15s\n",
                   registros[i].idRegistro,
                   registros[i].idFuncionario,
                   dataFormatada,
                   getTipoHoraString(registros[i].tipo), 
                   registros[i].horas,
                   registros[i].status);

            // <-- NOVO: Mostra a justificativa na lista do gestor também
            if (strcmp(registros[i].status, "Rejeitado") == 0 && strlen(registros[i].justificativa) > 0) {
                printf("      | Motivo: %s\n", registros[i].justificativa);
            }

            encontrados++;
        }
    }

    if (encontrados == 0) {
         printf("Nenhum registro encontrado com o status '%s'.\n", filtroStatus ? filtroStatus : "Qualquer");
    }
    printf("---------------------------------------------------------------------------------------------------\n");
}

/* (Ação do GESTOR) - MODIFICADO */
void aprovarReprovarHora() {
    int idBusca;
    char escolha;

    printf("\n--- Aprovar/Rejeitar Horas Pendentes ---\n");
    listarRegistros("Pendente");

    int pendentes = 0;
    for(int i=0; i < numRegistros; i++) {
        if(strcmp(registros[i].status, "Pendente") == 0) {
            pendentes++;
            break;
        }
    }
    if (pendentes == 0) return; 

    printf("\nDigite o ID do registro que deseja alterar: ");
    if (scanf("%d", &idBusca) != 1) {
         printf("ID inválido.\n");
         limparBufferEntrada();
         return;
    }
    limparBufferEntrada();

    int indiceEncontrado = -1;
    for (int i = 0; i < numRegistros; i++) {
        if (registros[i].idRegistro == idBusca) {
            indiceEncontrado = i;
            break;
        }
    }

    if (indiceEncontrado == -1) {
        printf("Erro: ID de registro %d não encontrado.\n", idBusca);
        return;
    }

    if (strcmp(registros[indiceEncontrado].status, "Pendente") != 0) {
        printf("Erro: O registro #%d não está 'Pendente' (Status: %s).\n", idBusca, registros[indiceEncontrado].status);
        return;
    }


    printf("Registro encontrado: [Funcionário: %s, Horas: %.1f, Tipo: %s, Status: %s]\n",
           registros[indiceEncontrado].idFuncionario,
           registros[indiceEncontrado].horas,
           getTipoHoraString(registros[indiceEncontrado].tipo),
           registros[indiceEncontrado].status);

    printf("Deseja (A)provar ou (R)ejeitar? (Qualquer outra tecla para cancelar): ");
    scanf(" %c", &escolha);
    limparBufferEntrada();

    if (escolha == 'A' || escolha == 'a') {
        strcpy(registros[indiceEncontrado].status, "Aprovado");
        strcpy(registros[indiceEncontrado].justificativa, ""); // <-- MODIFICADO: Limpa a justificativa
        printf("Registro %d APROVADO.\n", idBusca);
        salvarHoras(); 
    } else if (escolha == 'R' || escolha == 'r') {

        // --- Bloco NOVO ---
        char razao[150];
        printf("Digite a justificativa para a rejeição (max 150 chars):\n");
        lerString(razao, 150); // Usa a função de utilidade
        // --- Fim Bloco NOVO ---

        strcpy(registros[indiceEncontrado].status, "Rejeitado");
        strcpy(registros[indiceEncontrado].justificativa, razao); // <-- MODIFICADO: Salva a justificativa

        printf("Registro %d REJEITADO com motivo.\n", idBusca);
        salvarHoras(); 
    } else {
        printf("Ação cancelada.\n");
    }
}

/* (Ação do GESTOR) */
void calcularTotalHorasFuncionario() {
    char idFuncionario[50];
    float valorHoraNormal = 0;

    float totalAprovNormal = 0, totalAprovFDS = 0, totalAprovNormalNoturna = 0, totalAprovFDSNoturna = 0;
    float totalPendNormal = 0, totalPendFDS = 0, totalPendNormalNoturna = 0, totalPendFDSNoturna = 0;

    printf("\n--- Gerar Relatório Financeiro (Base Local) ---\n");
    printf("Digite o ID do Funcionário (ex: jose.silva): ");
    lerString(idFuncionario, 50);

    printf("Digite o valor da hora normal do funcionário (ex: 25.50): R$ ");
    while (scanf("%f", &valorHoraNormal) != 1 || valorHoraNormal <= 0) {
        printf("Entrada inválida. Digite um valor positivo (ex: 25.50): ");
        limparBufferEntrada();
    }
    limparBufferEntrada();

    for (int i = 0; i < numRegistros; i++) {
        if (strcmp(registros[i].idFuncionario, idFuncionario) == 0) {

            if (strcmp(registros[i].status, "Aprovado") == 0) {
                switch(registros[i].tipo) {
                    case TIPO_NORMAL: totalAprovNormal += registros[i].horas; break;
                    case TIPO_FDS_FERIADO: totalAprovFDS += registros[i].horas; break;
                    case TIPO_NORMAL_NOTURNA: totalAprovNormalNoturna += registros[i].horas; break;
                    case TIPO_FDS_NOTURNA: totalAprovFDSNoturna += registros[i].horas; break;
                }
            } else if (strcmp(registros[i].status, "Pendente") == 0) {
                switch(registros[i].tipo) {
                    case TIPO_NORMAL: totalPendNormal += registros[i].horas; break;
                    case TIPO_FDS_FERIADO: totalPendFDS += registros[i].horas; break;
                    case TIPO_NORMAL_NOTURNA: totalPendNormalNoturna += registros[i].horas; break;
                    case TIPO_FDS_NOTURNA: totalPendFDSNoturna += registros[i].horas; break;
                }
            }
            // Registros "Rejeitados" ou "Exportados" são ignorados no cálculo financeiro
        }
    }

    // --- Lógica de Cálculo Financeiro (CLT) ---
    float pagarNormal = totalAprovNormal * (valorHoraNormal * 1.50);
    float pagarFDS = totalAprovFDS * (valorHoraNormal * 2.00);
    float pagarNormalNoturna = totalAprovNormalNoturna * (valorHoraNormal * 1.80); 
    float pagarFDSNoturna = totalAprovFDSNoturna * (valorHoraNormal * 2.40); 
    float totalPagar = pagarNormal + pagarFDS + pagarNormalNoturna + pagarFDSNoturna;

    float pendNormal = totalPendNormal * (valorHoraNormal * 1.50);
    float pendFDS = totalPendFDS * (valorHoraNormal * 2.00);
    float pendNormalNoturna = totalPendNormalNoturna * (valorHoraNormal * 1.80);
    float pendFDSNoturna = totalPendFDSNoturna * (valorHoraNormal * 2.40);
    float totalPendente = pendNormal + pendFDS + pendNormalNoturna + pendFDSNoturna;

    printf("\n======================================================\n");
    printf("  Relatório Financeiro para: '%s'\n", idFuncionario);
    printf("  Valor Hora Normal: R$ %.2f\n", valorHoraNormal);
    printf("======================================================\n\n");

    printf("--- Horas APROVADAS (Total a Pagar: R$ %.2f) ---\n", totalPagar);
    printf("  Tipo Normal (50%%):         %5.2f horas = R$ %8.2f\n", totalAprovNormal, pagarNormal);
    printf("  FDS/Feriado (100%%):      %5.2f horas = R$ %8.2f\n", totalAprovFDS, pagarFDS);
    printf("  Normal Noturna (80%%):    %5.2f horas = R$ %8.2f\n", totalAprovNormalNoturna, pagarNormalNoturna);
    printf("  FDS Noturna (140%%):      %5.2f horas = R$ %8.2f\n", totalAprovFDSNoturna, pagarFDSNoturna);

    printf("\n--- Horas PENDENTES (Estimativa: R$ %.2f) ---\n", totalPendente);
    printf("  Tipo Normal (50%%):         %5.2f horas = R$ %8.2f\n", totalPendNormal, pendNormal);
    printf("  FDS/Feriado (100%%):      %5.2f horas = R$ %8.2f\n", totalPendFDS, pendFDS);
    printf("  Normal Noturna (80%%):    %5.2f horas = R$ %8.2f\n", totalPendNormalNoturna, pendNormalNoturna);
    printf("  FDS Noturna (140%%):      %5.2f horas = R$ %8.2f\n", totalPendFDSNoturna, pendFDSNoturna);
    printf("======================================================\n");
}


/* (Ação do GESTOR) - MODIFICADO */
void importarHorasCSV() {
    char nomeArquivo[100];
    char linha[512];
    int importados = 0;
    int duplicados = 0;
    int substituidos = 0; 

    printf("\n--- Importar Registros de Horas via CSV ---\n");
    printf("Digite o nome do arquivo .csv que deseja importar: ");
    lerString(nomeArquivo, 100);

    FILE *f = fopen(nomeArquivo, "r"); 
    if (f == NULL) {
        printf("Erro: Não foi possível abrir o arquivo '%s'.\n", nomeArquivo);
        printf("Verifique se o arquivo está na mesma pasta do programa.\n");
        return;
    }

    fgets(linha, 512, f); // Pula cabeçalho

    while (fgets(linha, 512, f) != NULL) {
        if (numRegistros >= MAX_REGISTROS) {
            printf("Aviso: Limite de registros locais atingido. Importação parada.\n");
            break;
        }

        RegistroHoraExtra tempReg;
        int tipoInt;
        long timestampLong;

        int camposLidos = sscanf(linha, "%d,%[^,],%ld,%f,%d,%19s", 
               &tempReg.idRegistro,
               tempReg.idFuncionario,
               &timestampLong,
               &tempReg.horas,
               &tipoInt,
               tempReg.status
        );

        if (camposLidos < 6) {
            continue; 
        }

        tempReg.timestamp = (time_t)timestampLong;
        tempReg.tipo = (TipoHora)tipoInt;
        strcpy(tempReg.justificativa, ""); // <-- NOVO: Inicializa justificativa

        if (strcmp(tempReg.status, "Pendente") != 0) {
            continue; 
        }

        int duplicado = 0;
        int indiceExistente = -1;
        for (int i = 0; i < numRegistros; i++) {
            if (registros[i].idRegistro == tempReg.idRegistro) {
                indiceExistente = i;
                if (strcmp(registros[i].status, "Exportado") != 0) {
                    duplicado = 1;
                }
                break;
            }
        }

        if (duplicado) {
            duplicados++;
        } else if (indiceExistente != -1) {
            registros[indiceExistente] = tempReg;
            substituidos++;
        } else {
            registros[numRegistros] = tempReg;
            if (tempReg.idRegistro >= proximoIdRegistro) {
                proximoIdRegistro = tempReg.idRegistro + 1;
            }
            numRegistros++;
            importados++;
        }
    }

    fclose(f);

    if (importados > 0 || substituidos > 0) {
        printf("\nImportação concluída.\n");
        printf("  %d novos registros 'Pendentes' foram adicionados.\n", importados);
        printf("  %d registros 'Exportados' foram atualizados para 'Pendente'.\n", substituidos);
        printf("  %d registros duplicados (mesmo ID) foram ignorados.\n", duplicados);
        salvarHoras(); 
    } else if (duplicados > 0) {
        printf("\nImportação concluída. Nenhum novo registro adicionado.\n");
        printf("  %d registros duplicados (mesmo ID) foram ignorados.\n", duplicados);
    } else {
        printf("\nNenhum registro 'Pendente' foi encontrado no arquivo '%s'.\n", nomeArquivo);
    }
}
