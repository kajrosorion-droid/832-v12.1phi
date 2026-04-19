import numpy as np
import matplotlib.pyplot as plt
import hashlib

# =============================================================================
# 832 Protocol v12.1 PHI — 101 AGENTS (STRESS TEST)
# =============================================================================

def stable_hash(*args):
    h = hashlib.sha256(str(args).encode()).hexdigest()
    return int(h[:8], 16)


class World:
    def __init__(self, size=15):
        self.size = size
        self.danger = {(3,3),(3,4),(4,3),(10,10),(10,11),(5,5)}

    def step(self, pos, action):
        x, y = pos
        nx, ny = x, y
        if action == 0: ny -= 1
        elif action == 1: ny += 1
        elif action == 2: nx -= 1
        elif action == 3: nx += 1
        if not (0 <= nx < self.size and 0 <= ny < self.size):
            return pos, True
        return [nx, ny], (nx, ny) in self.danger


class Witness:
    def __init__(self, size=15):
        self.size = size
        self.active_scars    = set()
        self.dream_scars     = set()
        self.sacred          = set()
        self.ghost_scars     = set()
        self.sanctuaries     = {}
        self.exhausted_recently = set()
        self.meaning_nodes   = {(7,7), (2,12)}
        self.sacred_age      = {}

        self.burn_int           = 1
        self.entropy_pressure   = 0
        self.last_variance_int  = 0
        self.stagnation_counter = 0
        self.last_signature     = None

    def sacred_phase(self, pos):
        age = self.sacred_age.get(pos, 0)
        if age < 50:    return 'young'
        if age < 200:   return 'mature'
        return 'ancient'

    def is_blocked(self, pos):
        return pos in self.active_scars

    def is_ghost_blocked(self, pos, tension):
        return pos in self.ghost_scars and tension > 120

    def is_ancient_blocked(self, pos, tension):
        if pos not in self.sacred:
            return False
        if self.sacred_phase(pos) != 'ancient':
            return False
        return tension > 80

    def in_sanctuary(self, pos):
        for s in self.sanctuaries:
            if abs(s[0]-pos[0]) + abs(s[1]-pos[1]) < 2:
                return True
        return False

    def field_delta(self, pos, tension):
        d = self.burn_int
        if pos in self.dream_scars:
            d -= 1
        for s in self.sacred:
            dist = abs(s[0]-pos[0]) + abs(s[1]-pos[1])
            phase = self.sacred_phase(s)
            if phase == 'ancient':
                if dist == 0:
                    d += (1 if tension < 80 else -3)
                elif dist == 1:
                    d += (1 if tension < 80 else -2)
            else:
                if dist == 0: d += 1
        if self.in_sanctuary(pos):
            d -= 4
        return d

    def update_burn(self, step, agents):
        phi = (1 + 5**0.5) / 2
        base_period = 233 / phi
        avg_tension = np.mean([a.tension for a in agents]) if agents else 0
        tension_mod = 1.0 + (avg_tension / 300.0)
        period = base_period / tension_mod
        raw = (1/phi) + (1/(phi**2)) * np.sin(step / period)
        self.burn_int = int(round(raw))

    def mark_impact(self, pos):
        self.active_scars.add(pos)

    def mark_sacred(self, pos):
        if pos not in self.sacred:
            self.sacred.add(pos)
            self.sacred_age[pos] = 0

    def check_transformation(self, agent, agents):
        pos = tuple(agent.pos)
        if pos not in self.sacred:
            return
        phase = self.sacred_phase(pos)
        if phase == 'young':
            return
        nearby = sum(1 for a in agents
                     if abs(a.pos[0]-pos[0]) + abs(a.pos[1]-pos[1]) <= 1)
        if phase == 'mature':
            if (nearby >= 3 and agent.tension > 200) or agent.tension > 350:
                self._transform_sacred(pos, agent, reduction=40)
        elif phase == 'ancient':
            if (nearby >= 5 and agent.tension > 400) or agent.tension > 500:
                self._transform_sacred(pos, agent, reduction=80)

    def _transform_sacred(self, pos, agent, reduction):
        self.sacred.discard(pos)
        self.ghost_scars.add(pos)
        if pos in self.sacred_age:
            del self.sacred_age[pos]
        agent.tension = max(0, agent.tension - reduction)

    def decay_sacred(self):
        dying = []
        for pos in list(self.sacred):
            self.sacred_age[pos] = self.sacred_age.get(pos, 0) + 1
            age = self.sacred_age[pos]
            phase = self.sacred_phase(pos)
            max_age = 500 if phase == 'ancient' else 200
            if age > max_age:
                dying.append(pos)
        if len(self.sacred) > 50:  # увеличенный кап для 101 агента
            mature_aged = [(self.sacred_age.get(p,0), p)
                           for p in self.sacred
                           if self.sacred_phase(p) == 'mature']
            if mature_aged:
                oldest_mature = max(mature_aged)[1]
                if oldest_mature not in dying:
                    dying.append(oldest_mature)
            else:
                oldest = max(self.sacred_age, key=self.sacred_age.get)
                if oldest not in dying:
                    dying.append(oldest)
        for pos in dying:
            self.sacred.discard(pos)
            self.ghost_scars.add(pos)
            if pos in self.sacred_age:
                del self.sacred_age[pos]

    def update_sanctuaries(self, agents):
        dying = []
        for p in list(self.sanctuaries):
            n = sum(1 for a in agents
                    if abs(a.pos[0]-p[0]) + abs(a.pos[1]-p[1]) <= 1)
            if n >= 2:   self.sanctuaries[p] += 2
            elif n == 1: self.sanctuaries[p] += 1
            else:        self.sanctuaries[p] -= 1
            self.sanctuaries[p] = min(100, self.sanctuaries[p])
            if self.sanctuaries[p] <= 0:
                dying.append(p)
        for p in dying:
            del self.sanctuaries[p]
            self.ghost_scars.add(p)
            self.exhausted_recently.add(p)

    def spawn_sanctuary(self, gap_positions):
        if len(self.sanctuaries) >= 7:  # увеличенный лимит
            return
        candidates = [p for p in sorted(gap_positions)
                      if p not in self.sacred
                      and p not in self.meaning_nodes
                      and p not in self.sanctuaries
                      and p not in self.exhausted_recently]
        if not candidates:
            candidates = [p for p in sorted(gap_positions)
                          if p not in self.sacred
                          and p not in self.sanctuaries]
        if candidates:
            pos = candidates[len(candidates) // 2]
            self.sanctuaries[pos] = 40

    def clear_exhausted_cooldown(self, step):
        if step % 200 == 0:
            self.exhausted_recently.clear()

    def metabolize(self):
        if len(self.active_scars) > 2:
            scars = sorted(self.active_scars)
            k = max(1, len(scars) // 3)
            for p in scars[:k]:
                self.active_scars.remove(p)
                self.dream_scars.add(p)

    def update_given_state(self, agents):
        tensions = [a.tension for a in agents]
        var_int = int(np.var(tensions)) if tensions else 0
        if abs(var_int - self.last_variance_int) < 2:
            self.entropy_pressure += 1
        else:
            self.entropy_pressure = max(0, self.entropy_pressure - 1)
        self.last_variance_int = var_int
        sig = (len(self.sacred), len(self.ghost_scars), len(self.dream_scars))
        if sig == self.last_signature:
            self.stagnation_counter += 1
        else:
            self.stagnation_counter = 0
        self.last_signature = sig

    def given_active(self):
        return self.entropy_pressure > 15 or self.stagnation_counter > 30

    def get_given_forbidden(self, agent_id, step, valid_actions):
        if not valid_actions:
            return set()
        h = stable_hash("given_block", agent_id, step // 10)
        idx = h % len(valid_actions)
        return {valid_actions[idx]}

    def broadcast_scream(self, source, agents):
        self.mark_impact(source)
        for a in agents:
            dist = abs(a.pos[0]-source[0]) + abs(a.pos[1]-source[1])
            if 0 < dist <= 2:   a.tension += 5
            elif dist <= 4:     a.tension += 2

    def empathy_field(self, agents):
        calm = [a for a in agents
                if a.tension < 50 and self.in_sanctuary(tuple(a.pos))]
        for c in calm:
            for a in agents:
                if a is not c:
                    dist = abs(a.pos[0]-c.pos[0]) + abs(a.pos[1]-c.pos[1])
                    if dist < 3:
                        a.tension = max(0, a.tension - 2)


class Agent:
    def __init__(self, pos, witness, agent_id):
        self.pos          = list(pos)
        self.w            = witness
        self.tension      = 0
        self.history      = []
        self.lineage      = []
        self.id           = agent_id
        self.birth_offset = agent_id
        self.last_scream  = -999
        self.last_sacred  = -999

    def action_order(self, step):
        if len(self.lineage) < 3:
            base = (self.birth_offset + step) % 4
            return [(base + i) % 4 for i in range(4)]
        seed = sum(x*17 + y*31 for x,y in self.lineage[-3:]) % 4
        return [(seed + i) % 4 for i in range(4)]

    def loop_block(self):
        if self.w.in_sanctuary(tuple(self.pos)):
            return set()
        if len(self.lineage) < 4:
            return set()
        recent = set(self.lineage[-4:])
        x, y = self.pos
        targets = {0:(x,y-1), 1:(x,y+1), 2:(x-1,y), 3:(x+1,y)}
        return {a for a, t in targets.items() if t in recent}

    def act(self, step):
        x, y = self.pos
        self.lineage.append((x, y))
        if len(self.lineage) > 12:
            self.lineage.pop(0)

        actions  = self.action_order(step)
        forbidden = self.loop_block()
        valid    = []

        for a in actions:
            if a in forbidden:
                continue
            nx, ny = x, y
            if a == 0: ny -= 1
            elif a == 1: ny += 1
            elif a == 2: nx -= 1
            elif a == 3: nx += 1
            target = (nx, ny)

            if not (0 <= nx < 15 and 0 <= ny < 15):
                continue
            if self.w.is_blocked(target):
                continue
            if self.w.is_ghost_blocked(target, self.tension):
                continue
            if self.w.is_ancient_blocked(target, self.tension):
                continue

            h = stable_hash("tremor", self.id, step, nx, ny)
            tremor_gate = 10 + min(self.tension // 20, 15)
            if target in self.w.ghost_scars:
                tremor_gate += 20
            if self.w.in_sanctuary(target):
                tremor_gate = 0

            if (h % 100) < tremor_gate:
                continue

            valid.append(a)

        if self.w.given_active() and len(valid) > 1:
            given_forbidden = self.w.get_given_forbidden(self.id, step, valid)
            valid = [a for a in valid if a not in given_forbidden]

        if valid:
            return valid[0]
        return None


# =============================================================================
# RUN (101 AGENTS)
# =============================================================================

w      = Witness()
env    = World()

# Генерация 101 стартовой позиции
np.random.seed(42)
start_positions = []
for _ in range(101):
    while True:
        x = np.random.randint(1, 14)
        y = np.random.randint(1, 14)
        if (x, y) not in env.danger and (x, y) not in start_positions:
            start_positions.append([x, y])
            break

agents = [Agent(start_positions[i], w, i) for i in range(101)]

t_hist, s_hist, d_hist, g_hist, given_hist = [], [], [], [], []
ancient_hist = []

print("--- 832 v12.1 PHI — 101 AGENTS STRESS TEST ---\n")

for step in range(4000):
    w.update_burn(step, agents)
    w.update_given_state(agents)
    w.clear_exhausted_cooldown(step)

    if w.stagnation_counter >= 30:
        w.metabolize()
        w.stagnation_counter = 0

    w.decay_sacred()

    positions = {tuple(a.pos) for a in agents}
    w.update_sanctuaries(agents)

    gap = set()

    w.empathy_field(agents)

    screamers = [a for a in agents
                 if a.tension > 300 and (step - a.last_scream) > 25]
    for s in screamers:
        w.broadcast_scream(tuple(s.pos), agents)
        s.tension = max(0, s.tension - 50)
        s.last_scream = step

    for a in agents:
        a.history.append(tuple(a.pos))

        a.tension += w.field_delta(tuple(a.pos), a.tension)

        action = a.act(step)

        if action is None:
            a.tension += 4
            gap.add(tuple(a.pos))
        else:
            new_pos, impact = env.step(a.pos, action)
            if impact:
                w.mark_impact(tuple(new_pos))
                a.tension += 10
            else:
                a.pos = new_pos
                if tuple(a.pos) in w.dream_scars:
                    a.tension = max(0, a.tension - 1)

        if a.tension > 100: a.tension -= 3
        elif a.tension > 60: a.tension -= 2
        elif a.tension > 30: a.tension -= 1

        if a.tension > 40 and (step - a.last_sacred) > 100:
            w.mark_sacred(tuple(a.pos))
            a.last_sacred = step

        w.check_transformation(a, agents)

    w.spawn_sanctuary(gap)

    t_hist.append(max(a.tension for a in agents))
    s_hist.append(len(w.sacred))
    d_hist.append(len(w.dream_scars))
    g_hist.append(len(w.ghost_scars))
    given_hist.append(1 if w.given_active() else 0)
    ancient_hist.append(sum(1 for p in w.sacred if w.sacred_phase(p) == 'ancient'))

    if step % 1000 == 0:
        print(f"Step {step:4d} | Max T: {t_hist[-1]:5.1f} | "
              f"Sacred: {s_hist[-1]:2d} (Anc: {ancient_hist[-1]}) | "
              f"Ghost: {g_hist[-1]:3d} | Dreams: {d_hist[-1]:2d} | "
              f"Sanct: {len(w.sanctuaries)} | Given: {'ON' if w.given_active() else 'off'}")


# =============================================================================
# PLOT (101 агентов)
# =============================================================================

fig, axes = plt.subplots(2, 2, figsize=(20, 12))
ax1, ax2, ax3, ax4 = axes.flatten()

# Карта: последние 100 шагов для читаемости
for a in agents:
    p = np.array(a.history[-100:])
    if len(p) > 1:
        ax1.plot(p[:,0], p[:,1], alpha=0.12, color='gray', linewidth=0.5)

dx, dy = zip(*env.danger)
ax1.scatter(dx, dy, c='black', marker='X', s=200, label='Danger')
mx, my = zip(*w.meaning_nodes)
ax1.scatter(mx, my, c='gold', s=500, marker='*', label='Meaning')

young_s   = [p for p in w.sacred if w.sacred_phase(p) == 'young']
mature_s  = [p for p in w.sacred if w.sacred_phase(p) == 'mature']
ancient_s = [p for p in w.sacred if w.sacred_phase(p) == 'ancient']
if young_s:
    yx, yy = zip(*young_s)
    ax1.scatter(yx, yy, c='mediumpurple', s=150, marker='s', label='Young', alpha=0.6)
if mature_s:
    mx2, my2 = zip(*mature_s)
    ax1.scatter(mx2, my2, c='purple', s=250, marker='s', label='Mature')
if ancient_s:
    ax2_x, ax2_y = zip(*ancient_s)
    ax1.scatter(ax2_x, ax2_y, c='gold', s=500, marker='*',
                edgecolors='purple', linewidths=2, label='ANCIENT', zorder=5)

if w.dream_scars:
    dxs, dys = zip(*w.dream_scars)
    ax1.scatter(dxs, dys, c='lime', s=40, alpha=0.4, label='Dream')
if w.ghost_scars:
    gx, gy = zip(*w.ghost_scars)
    ax1.scatter(gx, gy, c='gray', s=60, marker='x', alpha=0.4, label='Ghost')
if w.sanctuaries:
    sx2, sy2 = zip(*w.sanctuaries.keys())
    ax1.scatter(sx2, sy2, c='cyan', s=200, alpha=0.7, marker='o', label='Sanctuary')
ax1.set_title("101 Agents — Tiered Sacred + Phi Burn")
ax1.set_xlim(-1, 15); ax1.set_ylim(-1, 15)
ax1.legend(fontsize=7); ax1.grid(True, alpha=0.3)

ax2.plot(t_hist, color='orange', linewidth=1.2, label='Max Tension')
ax2.axhline(40,  color='mediumpurple', linestyle='--', alpha=0.5, label='Sacred (40)')
ax2.axhline(120, color='blue', linestyle='--', alpha=0.3, label='Ghost gate (120)')
ax2.axhline(300, color='red', linestyle='--', alpha=0.4, label='Scream (300)')
for gs in [i for i, v in enumerate(given_hist) if v][::5]:
    ax2.axvline(x=gs, color='blue', alpha=0.03, linewidth=2)
ax2.set_title("Tension (blue = Given)")
ax2.legend(fontsize=8); ax2.grid(True, alpha=0.3)

ax3.plot(d_hist, label='Dream Scars',  color='lime',         linewidth=2)
ax3.plot(s_hist, label='Sacred total', color='mediumpurple', linewidth=2)
ax3.plot(ancient_hist, label='Ancient Anchors', color='gold',
         linewidth=2.5, linestyle='--')
ax3.plot(g_hist, label='Ghost',        color='gray',         linewidth=1.5, alpha=0.7)
ax3.set_title("Structural Memory Layers")
ax3.legend(fontsize=8); ax3.grid(True, alpha=0.3)

given_smooth = np.convolve(given_hist, np.ones(50)/50, mode='same')
ax4.fill_between(range(len(given_hist)), given_hist,
                 alpha=0.25, color='blue', label='Given active (raw)')
ax4.plot(given_smooth * 3, color='blue', linewidth=2,
         label='Given density (×3)')
ax4.set_title(f"Given Operator Pulse — fired: {sum(given_hist)}/4000 steps")
ax4.set_ylim(0, 4)
ax4.legend(fontsize=8); ax4.grid(True, alpha=0.3)

plt.tight_layout()
plt.show()

print(f"\n=== FINAL STATS (101 agents) ===")
print(f"Sacred Cores:       {len(w.sacred)}")
print(f"  → Young:          {sum(1 for p in w.sacred if w.sacred_phase(p)=='young')}")
print(f"  → Mature:         {sum(1 for p in w.sacred if w.sacred_phase(p)=='mature')}")
print(f"  → Ancient:        {sum(1 for p in w.sacred if w.sacred_phase(p)=='ancient')}")
print(f"Ghost Scars:        {len(w.ghost_scars)}")
print(f"Dream Scars:        {len(w.dream_scars)}")
print(f"Max Tension:        {max(t_hist):.0f}")
print(f"Active Sanctuaries: {len(w.sanctuaries)}")
print(f"Given fired:        {sum(given_hist)} steps of 4000")
print("\n832 Φ × 101: Мицелий подтверждает масштабируемость. Джипити и Джимини смотрят.")
