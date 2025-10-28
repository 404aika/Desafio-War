
/* war_desafio_colored.c
   "Desafio War Estruturado – Tema 1"
   Versão: territorios já criados, cores ANSI, menu interativo, salvar/carregar, ataques animados simples
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h> // para sleep

// ANSI colors
#define BLUE "\x1b[34m"
#define RED "\x1b[31m"
#define GREEN "\x1b[32m"
#define YELLOW "\x1b[33m"
#define ORANGE "\x1b[38;5;208m"
#define PURPLE "\x1b[35m"
#define RESET "\x1b[0m"

typedef struct Territory {
    char *name;
    char *colorname;
    char *colorcode; // ANSI code
    int troops;
} Territory;

typedef struct Battle {
    Territory *attacker;
    Territory *defender;
    int attacker_losses;
    int defender_losses;
} Battle;

typedef struct Mission {
    char *desc;
    int type; // 1 = conquistar N territorios, 2 = eliminar cor, 3 = ter >= X tropas
    int target;
    char *color; // para type==2
} Mission;

/* duplicar string */
char *my_strdup(const char *s) {
    if (!s) return NULL;
    size_t n = strlen(s)+1;
    char *r = malloc(n);
    if (r) memcpy(r,s,n);
    return r;
}

/* vetor dinâmico de territories */
typedef struct TerritoryList {
    Territory *items;
    int size;
    int capacity;
} TerritoryList;

void tlist_init(TerritoryList *l) { l->items=NULL; l->size=l->capacity=0; }
void tlist_free(TerritoryList *l) {
    if(!l) return;
    for(int i=0;i<l->size;i++){
        free(l->items[i].name);
        free(l->items[i].colorname);
        free(l->items[i].colorcode);
    }
    free(l->items); l->items=NULL; l->size=l->capacity=0;
}
int tlist_resize(TerritoryList *l,int nc){
    Territory *tmp = realloc(l->items,sizeof(Territory)*nc);
    if(!tmp) return 0;
    l->items=tmp; l->capacity=nc; return 1;
}
int tlist_add(TerritoryList *l,const char *name,const char *colorname,const char *colorcode,int troops){
    if(l->size==l->capacity){
        int nc=(l->capacity==0)?4:l->capacity*2;
        if(!tlist_resize(l,nc)) return 0;
    }
    l->items[l->size].name=my_strdup(name);
    l->items[l->size].colorname=my_strdup(colorname);
    l->items[l->size].colorcode=my_strdup(colorcode);
    l->items[l->size].troops=troops;
    l->size++;
    return 1;
}
void tlist_print(TerritoryList *l){
    printf("Territórios cadastrados:\n");
    for(int i=0;i<l->size;i++){
        printf("[%d] %s%s%s (cor: %s) - Tropas: %d\n",
            i, l->items[i].colorcode, l->items[i].name, RESET,
            l->items[i].colorname,
            l->items[i].troops);
    }
}

/* missões */
typedef struct MissionList {
    Mission *items;
    int size, capacity;
} MissionList;
void mlist_init(MissionList *m){ m->items=NULL; m->size=m->capacity=0;}
void mlist_free(MissionList *m){
    if(!m) return;
    for(int i=0;i<m->size;i++) free(m->items[i].desc);
    free(m->items); m->items=NULL; m->size=m->capacity=0;
}
int mlist_resize(MissionList *m,int nc){
    Mission *tmp = realloc(m->items,sizeof(Mission)*nc);
    if(!tmp) return 0;
    m->items=tmp; m->capacity=nc; return 1;
}
int mlist_add(MissionList *m,Mission mm){
    if(m->size==m->capacity){
        int nc=(m->capacity==0)?4:m->capacity*2;
        if(!mlist_resize(m,nc)) return 0;
    }
    m->items[m->size++]=mm;
    return 1;
}
void mlist_print(MissionList *m){
    printf("Missões:\n");
    for(int i=0;i<m->size;i++){
        printf("[%d] %s\n",i,m->items[i].desc);
    }
}

/* ataques */
int cmpdesc(const void *a,const void *b){ return (*(int*)b)-(*(int*)a); }
Battle *battle_create(Territory *att,Territory *def,int attack_troops){
    Battle *b = malloc(sizeof(Battle));
    if(!b) return NULL;
    b->attacker=att; b->defender=def; b->attacker_losses=0; b->defender_losses=0;

    int atk_dice = attack_troops; if(atk_dice>3) atk_dice=3; if(atk_dice<1) atk_dice=1;
    int def_dice = def->troops; if(def_dice>2) def_dice=2; if(def_dice<1) def_dice=1;

    int *atk = malloc(sizeof(int)*atk_dice);
    int *df = malloc(sizeof(int)*def_dice);
    if(!atk||!df){ free(atk); free(df); free(b); return NULL; }

    for(int i=0;i<atk_dice;i++) atk[i]=rand()%6+1;
    for(int i=0;i<def_dice;i++) df[i]=rand()%6+1;
    qsort(atk,atk_dice,sizeof(int),cmpdesc);
    qsort(df,def_dice,sizeof(int),cmpdesc);

    int pairs = (atk_dice<def_dice)?atk_dice:def_dice;
    for(int i=0;i<pairs;i++){
        if(atk[i]>df[i]) b->defender_losses++;
        else b->attacker_losses++;
    }

    att->troops-=b->attacker_losses; if(att->troops<1) att->troops=1;
    def->troops-=b->defender_losses; if(def->troops<0) def->troops=0;

    free(atk); free(df); return b;
}

void battle_print(Battle *b){
    if(!b) return;
    printf("Batalha: %s (tropas %d) VS %s (tropas %d)\n",
        b->attacker->name, b->attacker->troops+b->attacker_losses,
        b->defender->name, b->defender->troops+b->defender_losses);
    printf("Perdas - Atacante: %d | Defensor: %d\n",
        b->attacker_losses, b->defender_losses);
}

/* salvar/carregar */
void save_game(TerritoryList *tlist,MissionList *mlist,const char *fname){
    FILE *f=fopen(fname,"w");
    if(!f){ printf("Erro ao salvar.\n"); return; }
    fprintf(f,"[TERRITORIES]\n");
    for(int i=0;i<tlist->size;i++)
        fprintf(f,"%s;%s;%s;%d\n",tlist->items[i].name,tlist->items[i].colorname,tlist->items[i].colorcode,tlist->items[i].troops);
    fprintf(f,"[MISSIONS]\n");
    for(int i=0;i<mlist->size;i++)
        fprintf(f,"%d;%d;%s\n",mlist->items[i].type,mlist->items[i].target,mlist->items[i].desc);
    fclose(f);
    printf("Jogo salvo em %s\n",fname);
}

void load_game(TerritoryList *tlist,MissionList *mlist,const char *fname){
    FILE *f=fopen(fname,"r");
    if(!f){ printf("Arquivo de save não encontrado.\n"); return; }
    tlist_free(tlist); mlist_free(mlist); tlist_init(tlist); mlist_init(mlist);
    char line[512]; char section[64]="";
    while(fgets(line,sizeof(line),f)){
        line[strcspn(line,"\n")=0];
        if(line[0]=='['){ sscanf(line,"[%63[^]]",section); continue;}
        if(strcmp(section,"TERRITORIES")==0){
            char name[128], colorname[64], colorcode[64]; int troops;
            if(sscanf(line,"%127[^;];%63[^;];%63[^;];%d",name,colorname,colorcode,&troops)==4)
                tlist_add(tlist,name,colorname,colorcode,troops);
        } else if(strcmp(section,"MISSIONS")==0){
            int typ,tgt; char desc[256];
            if(sscanf(line,"%d;%d;%255[^\n]",&typ,&tgt,desc)==3){
                Mission m; m.type=typ; m.target=tgt; m.color=NULL; m.desc=my_strdup(desc); mlist_add(mlist,m);
            }
        }
    }
    fclose(f);
    printf("Save carregado de %s\n",fname);
}

/* helpers */
int find_territory_by_name(TerritoryList *l,const char *name){
    for(int i=0;i<l->size;i++) if(strcmp(l->items[i].name,name)==0) return i;
    return -1;
}

/* menu */
void print_header(){ printf("=== Desafio War Estruturado – Tema 1 ===\n"); }
void print_menu(){
    printf("1 - Listar territórios\n");
    printf("2 - Realizar ataque\n");
    printf("3 - Criar missão\n");
    printf("4 - Mostrar missões\n");
    printf("5 - Salvar jogo\n");
    printf("6 - Carregar jogo\n");
    printf("0 - Sair\n");
    printf("> ");
}

int main(){
    srand((unsigned)time(NULL));
    TerritoryList tlist; tlist_init(&tlist);
    MissionList mlist; mlist_init(&mlist);

    // territorios já criados
    tlist_add(&tlist,"Território Azul","Azul",BLUE,5);
    tlist_add(&tlist,"Território Vermelho","Vermelho",RED,4);
    tlist_add(&tlist,"Território Verde","Verde",GREEN,3);
    tlist_add(&tlist,"Território Amarelo","Amarelo",YELLOW,2);
    tlist_add(&tlist,"Território Laranja","Laranja",ORANGE,3);
    tlist_add(&tlist,"Território Roxo","Roxo",PURPLE,2);

    int opt=-1; char line[256];
    while(1){
        print_header(); print_menu();
        if(scanf("%d",&opt)!=1){ int ch; while((ch=getchar())!=EOF && ch!='\n'); printf("Entrada inválida.\n"); continue; }
        int ch=getchar();
        if(opt==0){ tlist_free(&tlist); mlist_free(&mlist); printf("Saindo...\n"); return 0;}
        else if(opt==1) tlist_print(&tlist);
        else if(opt==2){
            tlist_print(&tlist);
            printf("Nome do território atacante: "); fgets(line,sizeof(line),stdin); line[strcspn(line,"\n")]=0;
            int ia=find_territory_by_name(&tlist,line); if(ia<0){ printf("Atacante não encontrado.\n"); continue;}
            printf("Nome do território defensor: "); fgets(line,sizeof(line),stdin); line[strcspn(line,"\n")]=0;
            int id=find_territory_by_name(&tlist,line); if(id<0){ printf("Defensor não encontrado.\n"); continue;}
            if(ia==id){ printf("Não pode atacar a si mesmo.\n"); continue;}
            Territory *att=&tlist.items[ia]; Territory *def=&tlist.items[id];
            if(att->troops<=1){ printf("Tropas insuficientes (>1 necessário).\n"); continue;}
            printf("Quantas tropas atacam? (1-3, max tropas-1): "); int atknum; if(scanf("%d",&atknum)!=1){ int ch; while((ch=getchar())!=EOF && ch!='\n'); printf("Entrada inválida.\n"); continue;}
            ch=getchar();
            if(atknum<1 || atknum>3 || atknum>att->troops-1){ printf("Número inválido.\n"); continue;}
            printf("⚔️ Atacando"); fflush(stdout); for(int i=0;i<3;i++){printf("."); fflush(stdout); sleep(1);} printf("\n");
            Battle *b=battle_create(att,def,atknum); if(!b){ printf("Erro na batalha.\n"); continue;}
            battle_print(b);
            if(def->troops==0){ printf("Defensor eliminado! %s conquista %s. Movendo 1 tropa.\n",att->name,def->name); def->troops=1; att->troops-=1;if(att->troops<1)att->troops=1;}
            free(b);
        }
        else if(opt==3){
            printf("Descrição da missão: "); fgets(line,sizeof(line),stdin); line[strcspn(line,"\n")]=0;
            if(strlen(line)==0){ printf("Missão vazia. Cancelada.\n"); continue;}
            printf("Tipo (1=conquistar N territorios; 2=eliminar cor; 3=ter >= X tropas): "); int typ; if(scanf("%d",&typ)!=1){ int ch; while((ch=getchar())!=EOF && ch!='\n'); printf("Entrada inválida.\n"); continue;} ch=getchar();
            Mission m; m.desc=my_strdup(line); m.type=typ; m.target=0; m.color=NULL;
            if(typ==1){ printf("N (territórios a ter com tropas>0): "); if(scanf("%d",&m.target)!=1){ int ch; while((ch=getchar())!=EOF && ch!='\n'); printf("Entrada inválida.\n"); free(m.desc); continue;} ch=getchar();}
            else if(typ==2){ char colbuf[128]; printf("Cor a eliminar: "); fgets(colbuf,sizeof(colbuf),stdin); colbuf[strcspn(colbuf,"\n")]=0; m.color=my_strdup(colbuf);}
            else if(typ==3){ printf("Total mínimo de tropas: "); if(scanf("%d",&m.target)!=1){ int ch; while((ch=getchar())!=EOF && ch!='\n'); printf("Entrada inválida.\n"); free(m.desc); continue;} ch=getchar();}
            else { printf("Tipo inválido. Cancelada.\n"); free(m.desc); continue;}
            mlist_add(&mlist,m); printf("Missão adicionada.\n");
        }
        else if(opt==4) mlist_print(&mlist);
        else if(opt==5) save_game(&tlist,&mlist,"save_war.txt");
        else if(opt==6) load_game(&tlist,&mlist,"save_war.txt");
        else printf("Opção inválida.\n");
    }
    return 0;
}
