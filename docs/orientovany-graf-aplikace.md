# Návrh aplikace pro výuku orientovaných ohodnocených grafů

## Architektura aplikace

### Přehled
- **Front-end**: Webová aplikace v Reactu (TypeScript) postavená na Vite pro rychlý vývoj.
- **Stavová logika**: React Query pro práci s daty z backendu a Zustand pro lehký globální stav (např. nastavení obtížnosti, profil hráče).
- **Vizualizace grafu**: Knihovna D3.js (skrze vlastní React komponenty) nebo React Flow pro interaktivní grafy.
- **Styly**: Tailwind CSS doplněné o vlastní komponenty a animace (Framer Motion).
- **Backend (volitelné)**: Node.js + Express nasazený na serverless platformě (např. Vercel). Slouží pro ukládání skóre, sdílených grafů a generování úloh.
- **Datové úložiště**: Firestore (nebo Supabase) pro perzistenci uživatelských profilů, skóre, sdílených grafů.
- **Autentizace**: Jednoduché přihlášení pomocí dětských přezdívek nebo rodičovského účtu (např. magic link).

### Front-end vrstvy
1. **Onboarding & výuková sekce**
   - Prezentace pojmů "uzel", "orientovaná hrana", "váha" pomocí krátkých animovaných scén.
   - Interaktivní mini-úkoly (klikni na šipku správného směru, porovnej dvě cesty).
2. **Výběr režimu hry**
   - Tři základní obtížnosti (Mini mapa, Dobrodružství, Expedice) + možnost vlastní mapy.
   - Nastavení pomocných prvků: nápovědy, časovač, zapnutí/ vypnutí zvuků.
3. **Herní plocha**
   - Canvas se zobrazeným grafem (uzly jako ostrovy, hrany jako mosty se šipkami a čísly).
   - Panel postupu: vybrané uzly, součet vah, dostupné nápovědy, čas.
   - Ovládací panel: tlačítka „Zpět“, „Zkontrolovat cestu“, „Nápověda“.
4. **Vyhodnocení**
   - Animované zvýraznění správné nejkratší cesty.
   - Tabulka srovnání: hráčova cesta vs. optimální cesta, rozdíl ve váze, bonusy.
   - Sdílení výsledku (kód pro kamaráda, leaderboard).

### Backendové služby (volitelné)
- **API /scores**: POST (uložení výsledku), GET (tabulka rekordů).
- **API /graphs**: GET (náhodný graf podle obtížnosti), POST (ukládání vlastních grafů).
- **API /users**: Správa hráčských profilů.
- **Generátor grafů**: Funkce, která vytváří orientované grafy s pozitivními vahami; zajišťuje dosažitelnost cílového uzlu.
- **Algoritmy**: Dijkstra pro výpočet nejkratší cesty, Floyd-Warshall pro předpočítané výsledky u menších grafů.

### Reprezentace dat
```ts
interface Graph {
  id: string;
  name: string;
  nodes: GraphNode[];
  edges: GraphEdge[];
  startNodeId: string;
  goalNodeId: string;
  requiredNodes?: string[]; // volitelná podmínka průchodu
}

interface GraphNode {
  id: string;
  label: string;
  position: { x: number; y: number };
}

interface GraphEdge {
  id: string;
  from: string;
  to: string;
  weight: number;
}
```

## Ukázkový front-end komponent
```tsx
import { useMemo, useState } from "react";
import { motion } from "framer-motion";

interface GraphProps {
  graph: Graph;
  onSubmitPath: (nodeIds: string[]) => void;
}

const GraphPlayground: React.FC<GraphProps> = ({ graph, onSubmitPath }) => {
  const [selectedPath, setSelectedPath] = useState<string[]>([graph.startNodeId]);
  const currentNodeId = selectedPath[selectedPath.length - 1];

  const outgoingEdges = useMemo(
    () => graph.edges.filter((edge) => edge.from === currentNodeId),
    [graph.edges, currentNodeId]
  );

  const handleNodeClick = (nodeId: string) => {
    const isReachable = outgoingEdges.some((edge) => edge.to === nodeId);
    if (!isReachable) return;
    setSelectedPath((prev) => [...prev, nodeId]);
  };

  const handleUndo = () => {
    if (selectedPath.length > 1) {
      setSelectedPath((prev) => prev.slice(0, -1));
    }
  };

  const handleSubmit = () => {
    onSubmitPath(selectedPath);
  };

  const totalWeight = selectedPath
    .slice(0, -1)
    .reduce((sum, nodeId, index) => {
      const nextId = selectedPath[index + 1];
      const edge = graph.edges.find((e) => e.from === nodeId && e.to === nextId);
      return edge ? sum + edge.weight : sum;
    }, 0);

  return (
    <div className="flex flex-col gap-4">
      <svg viewBox="0 0 800 600" className="w-full h-[400px] bg-sky-100 rounded-lg">
        {graph.edges.map((edge) => {
          const from = graph.nodes.find((node) => node.id === edge.from)!;
          const to = graph.nodes.find((node) => node.id === edge.to)!;

          return (
            <g key={edge.id}>
              <defs>
                <marker
                  id={`arrow-${edge.id}`}
                  markerWidth="10"
                  markerHeight="10"
                  refX="10"
                  refY="5"
                  orient="auto"
                >
                  <path d="M0,0 L10,5 L0,10 Z" fill="#2563eb" />
                </marker>
              </defs>
              <line
                x1={from.position.x}
                y1={from.position.y}
                x2={to.position.x}
                y2={to.position.y}
                stroke="#2563eb"
                strokeWidth={4}
                markerEnd={`url(#arrow-${edge.id})`}
              />
              <text
                x={(from.position.x + to.position.x) / 2}
                y={(from.position.y + to.position.y) / 2 - 10}
                textAnchor="middle"
                className="fill-blue-700 font-bold"
              >
                {edge.weight}
              </text>
            </g>
          );
        })}

        {graph.nodes.map((node) => {
          const isSelected = selectedPath.includes(node.id);
          const isCurrent = node.id === currentNodeId;
          return (
            <motion.g
              key={node.id}
              onClick={() => handleNodeClick(node.id)}
              className="cursor-pointer"
              animate={{ scale: isCurrent ? 1.1 : 1 }}
            >
              <circle cx={node.position.x} cy={node.position.y} r={32} fill={isSelected ? "#34d399" : "#60a5fa"} />
              <text x={node.position.x} y={node.position.y + 5} textAnchor="middle" className="fill-white font-bold text-lg">
                {node.label}
              </text>
            </motion.g>
          );
        })}
      </svg>

      <div className="flex items-center gap-3">
        <button onClick={handleUndo} className="px-4 py-2 bg-yellow-400 rounded-full font-semibold">
          Zpět
        </button>
        <button onClick={handleSubmit} className="px-4 py-2 bg-green-500 text-white rounded-full font-semibold">
          Odevzdat cestu
        </button>
        <span className="text-lg font-bold text-slate-700">Součet vah: {totalWeight}</span>
      </div>
    </div>
  );
};

export default GraphPlayground;
```

## Ukázkové úlohy
| Název | Popis grafu | Start → Cíl | Otázka | Nejkratší cesta (váha) |
|-------|--------------|-------------|--------|------------------------|
| Mosty na Ostrově Snů | 6 uzlů, jednoduché hodnoty 1–5 | A → F | Najdi nejkratší cestu z přístavu do majáku. | A → C → E → F (váha 7) |
| Dopravní dobrodružství | 8 uzlů, některé hrany mají váhu 8 kvůli zácpě | Start → Park | Vyhni se zácpě a doraz do parku. | Start → Ulice 2 → Park (váha 6) |
| Expedice přes džungli | 10 uzlů, nutno projít uzlem "Tábor" | Vesnice → Vodopád | Najdi nejkratší cestu, která prochází táborem. | Vesnice → Tábor → Jeskyně → Vodopád (váha 11) |
| Ledová výprava | 12 uzlů, některé hrany jednosměrné z kopce dolů | Igloo → Maják | Použij šipky správným směrem a najdi nejkratší cestu. | Igloo → Most → Přístav → Maják (váha 9) |
| Hvězdná závodní dráha | 7 uzlů, speciální zkratka s váhou 2 | Start → Cíl | Najdi tajnou zkratku pro nejrychlejší cestu. | Start → Planetka → Cíl (váha 4) |

## Design uživatelského rozhraní

### Barevná paleta
- **Primární modrá (ostrovy/pozadí)**: `#60A5FA`
- **Sekundární zelená (správné volby)**: `#34D399`
- **Akcent žlutý (upozornění, tlačítka)**: `#FACC15`
- **Neutrální text**: `#1F2937`
- **Pozadí panelů**: `#E0F2FE`

### Ikony a ilustrace
- Uzly jako roztomilé ostrovy s drobnými ilustracemi (město, hrad, strom).
- Hrany jako mosty s vlaječkami indikujícími směr.
- Ikony pro nápovědu (žárovka), časovač (písečné hodiny), hvězdy (odměny).

### Animace
- Jemné pulzování startovního uzlu a cíle.
- Zvýraznění hrany při přejetí myší.
- Po odevzdání správné cesty animované "rozsvícení" hrany a efekty hvězdiček.

### UX Flow
1. **Úvodní obrazovka**: tlačítko "Začít dobrodružství", možnost přihlásit se.
2. **Výuková sekce**: krátké komiksové vysvětlení, 3 interaktivní kroky.
3. **Výběr obtížnosti**: výběr mapy, nastavení pomocníků.
4. **Herní obrazovka**: vizualizace grafu, panel informací, tlačítka akcí.
5. **Vyhodnocení**: přehled výsledků, možnost hrát znovu, sdílet nebo pokračovat na vyšší úroveň.

### Zvukové efekty
- Krátké melodie při správném/špatném výběru hrany.
- Ambientní hudba s možností vypnutí.

### Responsivita
- Na tabletu graf zabírá většinu obrazovky, panely se přesouvají pod graf.
- Na mobilu graf ve scrollovatelné oblasti, ovládací prvky větší pro dotyk.

